# [从Yaml到Pod创建过程详解](https://github.com/ZHOUSH41/gitblog/issues/3)

> 这篇文章将会完整的叙述出以下内容：`从用户提交 yaml 文件到集群Pod 实际创建的过程`，剖析 K8s 核心机制，有助于读者理清 K8s 内部组件之间的交互逻辑

为了拆解从 Yaml 到 Pod 创建这个复杂的过程，文章分为三个章节来介绍：
1. 控制面： `API Server` 、`etcd `、`Admission Controller` 是如何协同工作的 
2. 调度与执行： 介绍 `Scheduler` 如何选择 node ，以及 `Kubelet` 和 `Container Runtime`  如何在 node 上拉起 Pod
3. 外挂系统：核心介绍 `Device Plugin` 是如何介入 Pod 创建流程

# 控制面
当你通过 kubectl 输入 `kubectl -kubeconfig=... apply -f pod.yaml` 之后，
1. API Sever 会收到创建 Pod 的请求。在收到创建 Pod请求之后，API server 首先会对请求进行鉴权，认证请求来源是谁，有没有权限创建 Pod。
2. 在完成鉴权之后会进入Admission Ctrl 阶段，Addmission Ctrl 会请求进行修改（mutation） 和验证（validate），值得注意的是 K8s 提供了 `dynamic admission contrl ` 扩展机制，方便用户自定义Admission 阶段，该过程如下：
	- A. 运行内置的 mutation ：比如 DefaultNamespace 插件，如果提交的请求namespace 为空，就修改成 default
	- B. 运行自定义的 mutation：API server 会查看集群里面注册的MutationWebhookConfiguration，把已经修改过的请求发给外部的Webhook服务，比如 Istio 就是在这里注入一个 sidecar
	- C. 运行内置的 validate
	- D. 运行自定义的validate：API server 会查看集群里面注册的ValidatingWebhookConfiguration，比如：自定义的validate不允许创建带某个 label，API server 就会拒绝这个请求
3. 持久化，经过程上述过程后，会把最后的 Pod 对象写入 etcd，此时 Pod 状态为 Pending

# 调度与执行
我们要创建的Pod 对象已经存储 etcd 中，但系统还不知道哪个 node 来运行这个 Pod。此时需要调度器（scheduler）登场发挥作用了。
## Scheduler
- **它是谁？** `Scheduler` (调度器) 中央调度员”。它通过informer 机制 watch `API Server`
- **它做什么？**
    1. **发现新工作**：它通过 `API Server` 发现 `etcd` 里出现了一个新的 Pod，这个 Pod 的 `spec.nodeName` 字段是空的
    2. **“海选” (Filtering - 过滤)**：`Scheduler` 会查看集群里所有的 Node。它会过滤掉不满足 Pod 要求的 Node。比如：
        - Pod 需要 2核 CPU，Node A 只剩 1核了（淘汰！）。
        - Pod 需要 GPU，Node B 根本没有 GPU（淘汰！）。
        - Pod 要求必须在“华东区”（有特定标签），Node C 在“华北区”（淘汰！）。
    3. **“打分” (Scoring - 排序)**：从“海选”通过的 Node 中（比如剩下了 Node D 和 Node E），`Scheduler` 会给它们打分。比如，Node D 负载很低（得分高），Node E 负载很高（得分低）。
    4. **最终决定**：`Scheduler` 选择得分最高的那个 Node（比如 Node D）。
    5. **更新订单**：它**不会**自己去联系 Node D。它只会回到 `API Server`，更新那个 Pod 对象的 `spec.nodeName` 字段，把它从“空”改成 “Node D”。
**注意**：`Scheduler` 只负责决策，不负责执行。它的结果只是修改了 etcd 里的记录，Pod 应该去 Node x
## Kubelet
- **它是谁？** 它是控制平面（Control Plane）安插在每个工作节点（Node）上的“代理人”。它也通过 informer 机制 watch `API Server`
- **它做什么？**
    1. **发现新任务**：在 `Scheduler` 调度完成之后。Node D 上的 `Kubelet` 发现 Pod spec. Nodename = Node D，
    2. **领任务**：`Kubelet` 立刻从 `API Server` 那里拿到这个 Pod 的完整定义
    3. **准备环境**：它开始准备 Pod 运行所需的一切，比如：
        - 创建数据卷（Volumes），这里可能会涉及 CSI 相关。
        - 如果需要特殊设备，需要device plugin，这点后面会讲到
        - （如果是第一次）通知 `Container Runtime` 去拉取镜像， CRI 相关。
        - 配置网络，这里需要CNI 插件。
    4. **下达执行命令**：`Kubelet` 不会自己去启动容器，它会把这个工作交给 Container Runtime。
**注意**：`Kubelet` 负责 1. 对 API-server 上报 Pod 状态，2. setup 环境，但具体运行容器并不是它的职责

## CSI（Container Storage Interface）
CSI 共有两个阶段，分别是 attach 和 Mount
- **Attach (附加) - 控制平面**：如果 Pod 用的是“网络存储”（比如 AWS EBS, 谷歌云盘），一个**在控制平面运行**的 `csi-controller` (它也在 Watch API Server) 看到 Pod 被调度到 Node C，它会立刻发出一个 API 请求，把那块“云盘”（Attach）到 Node C 这个虚拟机上。**这是 Kubelet 之外发生的**。
- **Mount (挂载) - Node 级别 (Kubelet 在此介入)**：
    - `Kubelet` 看到 Pod 的 `spec.volumes` 里定义了一个 `PersistentVolumeClaim` (PVC)。
    - 它**首先**会调用**本机**的 `CSI Node Plugin`（一个在 Node C 上的 `DaemonSet` Pod）：“嗨 CSI 插件，请把那个（刚 Attach 过来）的云盘，挂载到宿主机的一个特定目录，以便我稍后给 Pod 用。”
    - CSI 插件负责格式化（如果是新的）、`mount` 这个设备到宿主机上的一个临时路径（比如 `/var/lib/kubelet/pods/.../volumes/my-pvc`）。
    - CSI 插件完成后，告诉 `Kubelet`：“OK，地基打好了，路径是这个。”
## Container Runtime
- **它是谁？** 这就是真正在节点上跑容器的东西，比如 **Docker**、**containerd** 或 **CRI-O**。
- **它做什么？**
    1. `Kubelet` 通过一个标准接口 (CRI - Container Runtime Interface) 对 `Container Runtime` 说：“启动这个容器，使用这个镜像，挂载这些卷”。
    2. `Container Runtime` 负责拉取镜像（如果本地没有，如果本地有的话，会先走 cache）。
    3. 它负责使用 Linux 内核功能（如 Cgroups 和 Namespaces）来创建和启动容器。
    4. 容器**真正跑起来了**！
- **状态上报**：`Container Runtime` 把容器的状态（比如 "Running"）告诉 `Kubelet`。
- **`Kubelet` 再上报**：`Kubelet` 看到容器跑起来了，就高兴地跑去 `API Server` 那里更新 Pod 的状态（`status` 字段），改成 **Running**。
## CRI&CNI 运行时&网络
- **Kubelet 调用 CRI**：`Kubelet` 通过 CRI 接口对 `containerd` (或 Docker) 说：“启动这个 Pod Sandbox (沙箱)！”
    - **传递参数**：它会把**第 1 步**（CSI）拿到的存储路径（`/var/lib/kubelet/pods/.../volumes/my-pvc`）和**第 2 步**（Device Plugin）拿到的设备路径（`/dev/nvidia0`）一起传给 `Container Runtime`。
    - `Container Runtime` 会把这些路径**挂载 (bind mount)** 到容器内部。
- **Container Runtime 调用 CNI (网络)**：  
    **容器运行时调整 CNI（网络）** ：
    - `Container Runtime` 创建了容器的进程和命名空间（PID, Mount, ...），**但此时容器还没有 IP 地址**，它还在一个“网络隔离”的状态。
    - 此时，`Container Runtime`（根据 `Kubelet` 的配置）会调用**本机**上的 **CNI 插件**（比如 Calico, Flannel，它们通常也是 `DaemonSet`）。
    - `CNI` 插件执行以下操作： a. 从 CNI 的“IP 地址管理器”(IPAM) 那里申请一个 IP (比如 `10.244.2.5`)。 b. 在宿主机（Node C）上创建一个“虚拟网线对”(veth pair)。 c. 一端留在宿主机上（插到“交换机” - Linux Bridge 上），另一端插到容器的“网络命名空间”里，并配置上 `10.244.2.5` 这个 IP 地址和路由规则。
    - `CNI` 插件完成后，把分配的 IP 地址（`10.244.2.5`）返回给 `Container Runtime`。
- **启动！**
    - `Container Runtime` 看到网络也 OK 了，最后才启动容器的主进程（`CMD` / `ENTRYPOINT`）。
# 外挂系统
## Device Plugin
此刻，Pod 已经被调度上了正确的 Node 上，并且 kubelet 已经准备好启动它了，但是 Pod 订单上有一个特殊要求：
``` YAML
spec: 
  containers: 
  - name: cuda-container 
    image: nvidia/cuda:11.0-base 
    resources: 
      limits: 
        nvidia.com/gpu: 1
```
kubelet 自己并不知道 `nvidia.com/gpu` 是什么，它只认识 cpu、memory，所以这里需要解决两个问题：
1. scheduler需要知道Node 上有几块GPU 
2. 怎么Node 上的一块 GPU 分配给 Pod

这时候就需要 `Device Plugin` 的机制发挥作用了，它允许第三方厂商（比如 NVIDIA、Intel、Mellanox 等）来告诉 `Kubelet` 如何管理他们的特定硬件

具体的作用机制是：
- **1. 注册 (Registration)**
    1. 首先，集群管理员会在**每一个**需要特殊硬件的 Node 上（比如 Node C），以 `DaemonSet` 的形式部署一个 **Device Plugin Pod**（例如 `nvidia-device-plugin-ds`）。
    2. 这个 `nvidia-device-plugin` Pod 启动后，它会去**侦测**宿主机（Node C）上有多少块 NVIDIA GPU。假设它发现了 2 块 GPU（`/dev/nvidia0`, `/dev/nvidia1`）。
    3. 它通过一个 `gRPC` 接口（在 Node 本地，通常是一个 Unix socket 文件）主动**联系**本机的 `Kubelet`：“你好 `Kubelet`，我是 `nvidia.com/gpu` 插件，我在这里发现了 2 块 GPU 资源。”
- **2. Kubelet 向 API Server上报库存 (Advertising)**
    1. `Kubelet` 收到这个注册信息后，它立刻向 `API Server` **上报自己的状态 (Node Status)**。
    2. 它会在自己的 Node 对象里，更新一个叫做 `allocatable` (可分配资源) 的字段，在 `cpu` 和 `memory` 旁边，加上一行：`allocatable: { ... , nvidia.com/gpu: 2 }`。
    3. 现在，`API Server` 和 `etcd` 就知道了：“Node C 有 2 块 GPU 库存”。
- **3. 调度 (Scheduling)**
    1. 现在回到我们的第二站！当 `Scheduler` 看到那个要求 `nvidia.com/gpu: 1` 的 Pod 时，它会去查看所有 Node 的 `allocatable` 库存。
    2. 它会发现 Node C 有 `nvidia.com/gpu: 2`（库存 > 1），所以（在 Filtering 阶段）Node C 是一个合格的节点。
    3. `Scheduler` 于是把 Pod 分配给了 Node C。
- **4. 分配 (Allocation)**
    1. `Kubelet`（在 Node C 上）拿到了这个 Pod。它看到了 `nvidia.com/gpu: 1` 的请求。
    2. `Kubelet` **不会自己**去挑 GPU。它会再次通过 `gRPC` 联系 `nvidia-device-plugin` 分配具体的 GPU
    3. `nvidia-device-plugin` 负责管理 GPU 的“分配账本”。它说：“好的，我把 `/dev/nvidia0` 分配给 Pod A。” 然后它把这个设备路径（以及可能需要的环境变量或挂载）回复给 `Kubelet`。
    4. `Kubelet` 拿到了这个具体信息（比如“把 `/dev/nvidia0` 挂载进容器”）。
- **5. 启动 (Container Runtime)**
    1. `Kubelet` 最后去调用 `Container Runtime` (比如 `containerd`)。
    2. 启动这个 `nvidia/cuda:11.0-base` 镜像，并且，把宿主机的 `/dev/nvidia0` 设备**注入 (mount)** 到这个新容器的内部。
    3. `Container Runtime` 照做。    
**最终**
- **向 Kubelet 注册 (Register)**：告诉 `Kubelet` “我管理这种硬件”。
- **帮助 Kubelet 上报 (Advertise)**：让 `Kubelet` 告诉 `Scheduler` “我有多少这种硬件”。
- **替 Kubelet 分配 (Allocate)**：当 Pod 被调度过来后，告诉 `Kubelet` “请把_这块_具体的硬件给它”。

device plugin 这一设计，
1. 扩展性极强，让 K8s 可以“即插即用”地支持任何未来的新硬件
2. 插件管理分配，让插件开发者可以自由的虚拟化设备
3. 上报的功能，要求插件开发者需要管理健康检查

# 其他组件

## Kube-proxy
- **他是谁？** `kube-proxy` 负责的网络，特别是 `Service` 的实现
- **职责**：`kube-proxy` 的工作是确保 K8s 的 `Service` 能够正常的工作
- 它做什么？
	1. Watch API server，但它关心的是 `Service` 和 `Endpoints`
	2. 当一个新的 `Service` 被创建，或者一个 Pod 变得 "Running" 并且 "Ready"（`Kubelet` 会上报这个状态），这个 Pod 的 IP 就会被添加到 `Endpoints` 列表里。
	3. `kube-proxy` 看到这个变化后，就会在**它所在的 Node** 上修改网络规则（比如 `iptables` 或 `ipvs`）。
	4. 这些规则的作用是：“如果有人想访问 `Service` 的虚拟 IP（比如 `10.96.0.10`），我就把这个请求（做-网络地址转换-NAT）转发到它后面某个真实 Pod 的 IP（比如 `192.168.1.5`）上。”

# 一图胜千言
<img width="1468" height="776" alt="Image" src="https://github.com/user-attachments/assets/8e899a69-5732-4d2e-a9e0-f8cba0c3490b" />


> 题后记：我个人不太喜欢这种“概念化”的面试风格，原因是这些流程式的概念知道就是知道，不知道的话很难在面试的时间内通过某些线索去推敲出结论来，属于很难考察出来候选人思维的题目。但是，anyway，弄清楚这一问题确实帮助我理清了这里面的架构关系，仅此写一篇 blog 作为记录