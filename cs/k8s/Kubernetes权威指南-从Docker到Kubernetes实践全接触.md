# Kubernetes权威指南：从Docker到Kubernetes实践全接触

## 第一章 Kubernetes入门

每个Pod中都运行着一个特殊的被称为Pause的容器，其他容器则为业务容器，这些业务容器共享Pause容器的网络栈和Volume挂载卷

k8s将集群中的机器划分为一个Master和一些Node
在Master上运行着集群管理相关的一组进程：kube-apiserver、kube-controller-manager和kube-scheduler
在Node上运行着kubelet、kube-proxy服务进程、Docker Engine等，这些服务进程负责Pod的创建、启动、监控、重启、销毁，以及实现软件模式的负载均衡器

Kubernetes要求底层网络支持集群内任意两个Pod之间的TCP/IP直接通信，这通常采用虚拟二层网络技术来实现（例如Flannel，Open vSwitch等）

Pod Volume是被定义在Pod上，然后被各个容器挂载到自己的文件系统中的

通常，我们会把Requests设置为一个较小的数值，符合容器平时的工作负载情况下的资源需求，而把Limit设置为峰值负载情况下资源占用的最大值。

HPA(Horizontal Pod Autoscaling即Pod横向自动扩容)：也属于一种k8s资源对象

Headless Service与普通Service的关键区别在于，它没有Cluster IP，如果解析Headless Service的DNS域名，则返回的是该Service对应的全部Pod的Endpoint列表（常见与StatefulSet配合使用）

运行在每个Node上的kube-proxy进程其实就是一个智能的软件负载均衡器，负责把对Service的请求转发到后端的某个Pod实例上，并在内部实现服务的负载均衡与会话保持机制

k8s的服务发现机制： Linux环境变量 -> DNS系统

### Volume

Volume是Pod中能够被多个容器访问的共享目录。k8s的volume概念、用途和目的与Docker的volume比较类似，但两者不能等价。首先，k8s中的volume被定义扎起pod上，然后被一个pod里的多个容器挂载到具体的文件目录下；其次，k8s中的volume与pod的生命周期相同，但与容器的生命周期不相关，当容器终止或者重启时，volume中的数据也不会丢失

volume类型：
 - emptyDir: Pod分配到Node时创建的。初始内容为空，并且无须指定宿主机上对应的目录文件，因为这是Kubernetes自动分配的一个目录，当Pod从Node上移除时，emptyDir中的数据也会被永久删除（注意：目前无法控制emptyDir使用的介质种类（硬盘，固态硬盘，内存）。）
 - hostPath：在Pod上挂载宿主机上的文件或目录
 - gcePersistentDisk：使用谷歌公有云提供的永久磁盘（Persistent Disk，PD）存放Volume的数据，它与emptyDir不同，PD上的内容会被永久保存，当Pod被删除时，PD只是被卸载（Unmount），但不会被删除。需要注意的是，你需要先创建一个PD，才能使用gcePersistentDisk。
 - awsElasticBlockStore：亚马逊公有云提供的EBS Volume存储数据，需要先创建一个EBS Volume才能使用awsElasticBlockStore
 - NFS：使用NFS网络文件系统提供的共享目录存储数据时，我们需要在系统中部署一个NFS Server
 - 其他：iscsi,flocker,glusterfs,rbd,gitRepo,secret，configMap

### Persistent Volume

Volume是被定义在Pod上的，属于计算资源的一部分，而实际上，网络存储是相对独立于计算资源而存在的一种实体资源。比如在使用虚拟机的情况下，我们通常会先定义一个网络存储，然后从中划出一个“网盘”并挂接到虚拟机上。Persistent Volume（PV）和与之相关联的Persistent Volume Claim（PVC）也起到了类似的作用。

PV可以被理解成Kubernetes集群中的某个网络存储对应的一块存储，它与Volume类似，但有以下区别。

- PV只能是网络存储，不属于任何Node，但可以在每个Node上访问。
- PV并不是被定义在Pod上的，而是独立于Pod之外定义的。
- PV目前支持的类型包括：gcePersistentDisk、AWSElasticBlockStore、AzureFile、AzureDisk、FC（Fibre Channel）、Flocker、NFS、iSCSI、RBD（Rados BlockDevice）、CephFS、Cinder、GlusterFS、VsphereVolume、Quobyte Volumes、VMware Photon、Portworx Volumes、ScaleIO Volumes和HostPath（仅供单机测试）。

## 第二章 Kubernetes安装配置指南

k8s支持的容器运行时(CRI)：Docker，Containerd，CRI-O和frakti

安装工具：kubeadm

master上的服务：etcd,kube-apiserver,kube-controller-manager,kube-scheduler
node上的服务：kubelet，kube-proxy，Docker(或其他容器运行时)

kubelet的职责在于通过RPC（CRI）管理容器的生命周期，实现容器生命周期的钩子，存活和健康监测，以及执行Pod的重启策略等

## 第三章 深入掌握Pod

### 静态pod

静态pod：由kubelet进行管理的仅存在于特定Node上的Pod。不能通过API Server进行管理，无法与RC，Deployment，DaemonSet进行关联，并且kubelet无法对他们进行健康检查

创建方式：配置文件方式和HTTP方式

### ConfigMap

容器应用对ConfigMap的使用有以下两种方法：

- 通过环境变量获取ConfigMap中的内容
- 通过Volume挂载的方式将ConfigMap中的内容挂载为容器内部的文件或目录

### Downward API

通过Downward API可以在容器内获取Pod信息

Downward API可以通过以下两种方式将Pod信息注入容器内部。

- （1）环境变量：用于单个变量，可以将Pod信息和Container信息注入容器内部。
- （2）Volume挂载：将数组类信息生成为文件并挂载到容器内部。

Downward API的价值：在某些集群中，集群中的每个节点都需要将自身的标识（ID）及进程绑定的IP地址等信息事先写入配置文件中，进程在启动时会读取这些信息，然后将这些信息发布到某个类似服务注册中心的地方，以实现集群节点的自动发现功能。此时Downward API就可以派上用场了，具体做法是先编写一个预启动脚本或Init Container，通过环境变量或文件方式获取Pod自身的名称、IP地址等信息，然后将这些信息写入主程序的配置文件中，最后启动主程序。

### Pod调度

配置调度策略：在Pod的定义中使用NodeSelector、NodeAffinity、PodAffinity、Pod驱逐（Taints和Tolerations（污点和容忍））等更加细粒度的调度策略设置，就能完成对Pod的精准调度

Pod Priority Preemption：Pod优先级调度。优先级抢占调度策略的核心行为分别是驱逐（Eviction）与抢占（Preemption），相关的资源有PriorityClasses。使用优先级抢占的调度策略可能会导致某些Pod永远无法被成功调度。因此优先级调度不但增加了系统的复杂性，还可能带来额外不稳定的因素。因此，一旦发生资源紧张的局面，首先要考虑的是集群扩容，如果无法扩容，则再考虑有监管的优先级调度特性，比如结合基于Namespace的资源配额限制来约束任意优先级抢占行为。

### Init Controller

### Pod的扩缩容

手动：

- 执行kubectl scale命令
- 通过RESTful API对一个Deployment/RC进行Pod副本数量的设置

自动：根据某个性能指标或者自定义业务指标，并指定Pod副本数量的范围，系统将自动在这个范围内根据性能指标的变化进行调整（Horizontal Pod Autoscaler（HPA）的控制器 + HorizontalPodAutoscaler资源对象 + Metrics Sever + scale操作）

自动扩容可以使用Prometheus来监控各项数据

## 第四章 深入掌握Service

Service的类型: ClusterIP,NodePort,LoadBalancer

Service的两种负载分发策略：

- RoundRobin: 轮询模式，即轮询将请求转发到后端的各个Pod上
- SessionAffinity：基于客户端IP地址进行会话保持模式

### Headless Service

在某些应用场景中，开发人员希望自己控制负载均衡的策略，不使用Service提供的默认负载均衡的功能，或者应用程序希望知道属于同组服务的其他实例。Kubernetes提供了HeadlessService来实现这种功能，即不为Service设置ClusterIP（入口IP地址），仅通过Label Selector将后端的Pod列表返回给调用的客户端。

配置中clusterIP为None

### 从集群外部访问Pod或Service

1. 将容器应用的端口号映射到物理机(Pod资源配置)
  - 通过设置容器级别的hostPort，将容器应用的端口号映射到物理机上
  - 通过设置Pod级别的hostNetwork=true，该Pod中所有容器的端口号都将被直接映射到物理机上

2. 将Service的端口号映射到物理机(Service资源配置)
  - 通过设置nodePort映射到物理机，同时设置Service的类型为NodePort
  - 通过设置LoadBalancer映射到云服务商提供的LoadBalancer地址。这种用法仅用于在公有云服务提供商的云平台上设置Service的场景

### DNS服务

DNS服务部署好后，需要将所有Node上的kubelet启动参数--cluster-dns设置为这个ClusterIP

除了使用集群范围的DNS服务(如CoreDNS)，在Pod级别也能设置DNS相关的策略和配置：dnsPolicy，dnsConfig

### Ingress

Ingress也是一种资源对象，配合一个具体的Ingress Controller使用，两者结合并实现了一个完整的Ingress负载均衡器

使用Ingress进行负载分发时，Ingress Controller基于Ingress规则将客户端请求直接转发到Service对应的后端Endpoint（Pod）上，这样会跳过kube-proxy的转发功能，kube-proxy不再起作用。如果Ingress Controller提供的是对外服务，则实际上实现的是边缘路由器的功能。

Ingress Controller实际上也是以pod的形式来运行的

使用Nginx来实现一个Ingress Controller的基本逻辑
1. 监听API Server，获取全部Ingress的定义。
2. 基于Ingress的定义，生成Nginx所需的配置文件/etc/nginx/nginx.conf。
3. 执行nginx -s reload命令，重新加载nginx.conf配置文件的内容。

## 第五章 核心组件运行机制

### Kubernetes API Server

Kubernetes API Server通过一个名为kube-apiserver的进程提供服务，该进程运行在Master上

如果只想对外暴露部分REST服务，则可以在Master或其他节点上运行kubectl proxy进程启动一个内部代理来实现

Kubernetes API Server本身也是一个Service，它的名称就是kubernetes，并且它的Cluster IP地址是Cluster IP地址池里的第1个地址

为了让Kubernetes中的其他组件在不访问底层etcd数据库的情况下，也能及时获取资源对象的变化事件，API Server模仿etcd的Watch API接口提供了自己的Watch接口，这样一来，这些组件就能近乎实时地获取它们感兴趣的任意资源对象的相关事件通知了(例如提供给controller-manager、scheduler、kublet等组件)

Kubernetes Proxy API接口，这类接口的作用是代理REST请求，即Kubernetes API Server把收到的REST请求转发到某个Node上的kubelet守护进程的REST端口，由该kubelet进程负责响应。在Kubernetes集群之外访问某个Pod容器的服务（HTTP服务）时，可以用Proxy API实现，这种场景多用于管理目的，比如逐一排查Service的Pod副本，检查哪些Pod的服务存在异常。

### Controller Manager

在Kubernetes集群中，每个Controller类似于“操作系统”，它们通过API Server提供的（List-Watch）接口实时监控集群中特定资源的状态变化，当发生各种故障导致某资源对象的状态发生变化时，Controller会尝试将其状态调整为期望的状态

Controller Manager内部包含（每种Controller都负责一种特定资源的控制流程）
- Replication Controller
- Node Controller
- ResourceQuota Controller
- Namespace Controller
- ServiceAccount Controller
- TokenController
- Service Controller
- Endpoint Controller

### Scheduler

Kubernetes Scheduler的作用是将待调度的Pod（API新创建的Pod、ControllerManager为补足副本而创建的Pod等）按照特定的调度算法和调度策略绑定（Binding）到集群中某个合适的Node上，并将绑定信息写入etcd中。

随后，目标节点上的kubelet通过API Server监听到Kubernetes Scheduler产生的Pod绑定事件，然后获取对应的Pod清单，下载Image镜像并启动容器。

默认调度流程
1. 预选调度过程，即遍历所有目标Node，筛选出符合要求的候选节点（可以配置预选策略）
2. 确定最优节点，在第1步的基础上，采用优选策略（xxx Priority）计算出每个候选节点的积分，积分最高者胜出。

### kubelet

在每个Node（又称Minion）上都会启动一个kubelet服务进程。该进程用于处理Master下发到本节点的任务，管理Pod及Pod中的容器。

kubelet通过以下几种方式获取自身Node上要运行的Pod清单(所有以非API Server方式创建的Pod都叫作Static Pod)
- 文件：kubelet启动参数“--config”指定的配置文件目录下的文件（默认目录为“/etc/kubernetes/manifests/”）
- HTTP端点（URL）：通过“--manifest-url”参数设置
- API Server：kubelet通过API Server监听etcd目录，同步Pod列表

kubelet定期调用容器中的LivenessProbe探针来诊断容器的健康状况

### kube-proxy

在Kubernetes集群的每个Node上都会运行一个kube-proxy服务进程，我们可以把这个进程看作Service的透明代理兼负载均衡器，其核心功能是将到某个Service的访问请求转发到后端的多个Pod实例上

## 第六章 深入分析集群安全机制

主要包括Authentication、Authorization、Admission Control、Secret和Service Account等方面

### API Server认证管理

客户端身份认证方式的三种方式
- 最严格的HTTPS证书认证：基于CA根证书签名的双向数字证书认证方式。
- HTTP Token认证：通过一个Token来识别合法用户。
- HTTP Base认证：通过用户名+密码的方式认证。

### API Server授权管理

API Server目前支持以下几种授权策略（通过API Server的启动参数“--authorization-mode”设置）
- AlwaysDeny：表示拒绝所有请求，一般用于测试。
- AlwaysAllow：允许接收所有请求，如果集群不需要授权流程，则可以采用该策略，这也是Kubernetes的默认配置。
- ABAC（Attribute-Based Access Control）：基于属性的访问控制，表示使用用户配置的授权规则对用户请求进行匹配和控制。
- Webhook：通过调用外部REST服务对用户进行授权。
- RBAC：Role-Based Access Control，基于角色的访问控制。
- Node：是一种专用模式，用于对kubelet发出的请求进行访问控制。

RBAC方式相关的资源对象：Role 、 ClusterRole 、 RoleBinding和ClusterRoleBinding

角色绑定中的用户来源：User，Group，ServiceAccount

API Server会创建一套默认的ClusterRole和ClusterRoleBinding对象，其中很多是以“system:”为前缀的，以表明这些资源属于基础架构，对这些对象的改动可能造成集群故障

### Admission Control（准入控制）

在API Server上设置参数即可定制我们需要的准入控制链（以插件的方式）

### Service Account

Service Account也是一种账号，但它并不是给Kubernetes集群的用户（系统管理员、运维人员、租户用户等）用的，而是给运行在Pod里的进程用的，它为Pod里的进程提供了必要的身份证明

在Pod中访问Kubernetes API Server服务时，是以Service方式访问名为Kubernetes的这个服务的

Service Accoun对象中通常包含了Secret对象（一个或多个），名为Tokens的Secret对象可以当作Volume被挂载到Pod的指定目录下，用来协助完成Pod中的进程访问API Server时的身份鉴权

Service Account的正常工作离不开以下几个控制器
- Admission Controller
- Token Controller。
- ServiceAccount Controller。

### Secret

创建Secret后的使用方式
- 在创建Pod时，通过为Pod指定Service Account来自动使用该Secret。
- 通过挂载该Secret到Pod来使用它。
- 在Docker镜像下载时使用，通过指定Pod的spc.ImagePullSecrets来引用它。

为了使用更新后的Secret，必须删除旧Pod，并重新创建一个新Pod

### Pod的安全策略配置

PodSecurityPolicy资源对象
admission-plugins：PodSecurityPolicy

在配置了PodSecurityPolicy后，对Pod操作时就会进行相应的准入控制判断

同时，Pod和容器的安全策略可以在Pod或Container的securityContext字段中进行设置

## 第七章 网络原理

Kubernetes网络模型设计的一个基础原则是：每个Pod都拥有一个独立的IP地址，并假定所有Pod都在一个可以直接连通的、扁平的网络空间中。(IP-per-Pod模型)

### Docker网络基础

#### 网络命名空间（Network Namespace）

在Linux的网络命名空间中可以有自己独立的路由表及独立的iptables设置来提供包转发、NAT及IP包过滤等功能。

Veth设备对：让处于不同命名空间的网络相互通信，甚至和外部的网络进行通信。Veth设备对的一个重要作用就是打通互相看不到的协议栈之间的壁垒，它就像一条管子，一端连着这个网络命名空间的协议栈，一端连着另一个网络命名空间的协议栈。

#### 网桥

网桥：网桥是一个二层的虚拟网络设备，把若干个网络接口“连接”起来，以使得网络接口之间的报文能够互相转发（在Linux中，一个网口其实就是一个物理网卡）

#### iptables和Netfilter

在Linux网络协议栈中有一组回调函数挂接点，通过这些挂接点挂接的钩子函数可以在Linux网络栈处理数据包的过程中对数据包进行一些操作，例如过滤、修改、丢弃等。整个挂接点技术叫作Netfilter和iptables。

Netfilter负责在内核中执行各种挂接的规则，运行在内核模式中；而iptables是在用户模式下运行的进程，负责协助和维护内核中Netfilter的各种规则表。二者互相配合来实现整个Linux网络协议栈中灵活的数据包处理机制。

#### Docker的网络实现

标准的Docker支持以下4类网络模式：host模式，container模式，none模式，bridge模式
在Kubernetes管理模式下通常只会使用bridge模式（Docker0网桥+Veth设备对）

#### Kubernetes的网络实现

容器到容器的通信：同一个Pod内的容器（Pod内的容器是不会跨宿主机的）共享同一个网络命名空间，共享同一个Linux协议栈。所以对于网络的各类操作，就和它们在同一台机器上一样，它们甚至可以用localhost地址访问彼此的端口。

Pod之间的通信：
- 同一个Node内Pod之间的通信：多个Pod都是通过Veth连接到同一个docker0网桥上
- 不同Node上Pod之间的通信：
  - Kubernetes会记录所有正在运行的Pod的IP分配信息，并将这些信息保存在etcd中（作为Service的Endpoint）
  - 在谷歌的GCE环境中，Pod的IP管理（类似docker0）、分配及它们之间的路由打通都是由GCE完成的。Kubernetes作为主要在GCE上面运行的框架，它的设计是假设底层已经具备这些条件，所以它分配完地址并将地址记录下来就完成了它的工作。在实际的GCE环境中，GCE的网络组件会读取这些信息，实现具体的网络打通。
  - 在实际的私有云环境中，除了需要部署Kubernetes和Docker，还需要额外的网络配置，甚至通过一些软件来实现Kubernetes对网络的要求

service的IP段可以是任何段（在master上启动API服务进程时，使用--service-cluster-ip-range=xx命令行参数指定），只要不和docker0或者物理网络的子网冲突就可以

#### CNI网络模型

Docker公司提出的Container Network Model（CNM）模型
CoreOS公司提出的Container NetworkInterface（CNI）模型，CNI定义的是容器运行环境与网络插件之间的简单接口规范

#### Kubernetes网络策略

资源对象：NetworkPolicy
策略控制器（Policy Controller）：policy controller需要实现一个API Listener，监听用户设置的NetworkPolicy定义，并将网络访问规则通过各Node的Agent进行实际设置（Agent则需要通过CNI网络插件实现）

#### 开源的网络组件

Flannel、Open vSwitch、直接路由和Calico

## 第八章 共享存储原理

Kubernetes从1.9版本开始引入容器存储接口Container Storage Interface（CSI）机制，目标是在Kubernetes和外部存储系统之间建立一套标准的存储管理接口，通过该接口为容器提供存储服务，类似于CRI（容器运行时接口）和CNI（容器网络接口）。

主要资源对象：PV、PVC、StorageClass

### PV

PV作为存储资源，主要包括存储能力、访问模式、存储类型、回收策略、后端存储类型等关键信息的设置。

某个PV在生命周期中可能处于以下4个阶段（Phaes）之一
- Available：可用状态，还未与某个PVC绑定。
- Bound：已与某个PVC绑定
- Released：绑定的PVC已经删除，资源已释放，但没有被集群回收
- Failed：自动资源回收失败

### PVC

PVC作为用户对存储资源的需求申请，主要包括存储空间请求、访问模式、PV选择条件和存储类别等信息的设置

注意，PVC和PV都受限于Namespace，PVC在选择PV时受到Namespace的限制，只有相同Namespace中的PV才可能与PVC绑定。Pod在引用PVC时同样受Namespace的限制，只有相同Namespace中的PVC才能挂载到Pod内。

Kubernetes支持两种资源的供应模式：静态模式（Static）和动态模式（Dynamic）。资源供应的结果就是创建好的PV。
- 静态模式：集群管理员手工创建许多PV，在定义PV时需要将后端存储的特性进行设置
- 动态模式：集群管理员无须手工创建PV，而是通过StorageClass的设置对后端存储进行描述，标记为某种类型。此时要求PVC对存储的类型进行声明，系统将自动完成PV的创建及与PVC的绑定。PVC可以声明Class为""，说明该PVC禁止使用动态模式

### StorageClass

StorageClass作为对存储资源的抽象定义，对用户设置的PVC申请屏蔽后端存储的细节，一方面减少了用户对于存储资源细节的关注，另一方面减轻了管理员手工管理PV的工作，由系统自动完成PV的创建和绑定，实现了动态的资源供应。基于StorageClass的动态资源供应模式将逐步成为云平台的标准存储配置模式。

要在系统中设置一个默认的StorageClass，则首先需要启用名为DefaultStorageClass的admission controller，即在kube-apiserver的命令行参数--admission-control中增加
```
--admission-control=...,DefaultStorageClass
```
在StorageClass的定义中设置一个annotation
```
storageclass.beta.kubernetes.io/is-default-class="true"
```

### CSI存储机制详解

Kubernetes从1.9版本开始引入容器存储接口Container Storage Interface（CSI）机制，用于在Kubernetes和外部存储系统之间建立一套标准的存储管理接口，通过该接口为容器提供存储服务

存储提供方只需要基于标准接口进行存储插件的实现，就能使用Kubernetes的原生存储机制为容器提供存储服务。

每种CSI存储插件都提供了容器镜像，与external-attacher、external-provisioner、node-driver-registrar等sidecar辅助容器共同完成存储插件系统的部署，每个插件的部署配置详见官网https://kubernetes-csi.github.io/docs/drivers.html中的链接

## 第9章 Kubernetes开发指南

### Kubernetes API

### Kubernetes API的扩展

目前Kubernetes提供了以下两种机制供用户扩展API
- 使用CRD机制：复用Kubernetes的API Server，无须编写额外的API Server。用户只需要定义CRD，并且提供一个CRD控制器，就能通过Kubernetes的API管理自定义资源对象了，同时要求用户的CRD对象符合API Server的管理规范。
- 使用API聚合机制：用户需要编写额外的API Server，可以对资源进行更细粒度的控制（例如，如何在各API版本之间切换），要求用户自行处理对多个API版本的支持。

## 第10章 Kubernetes集群管理

Node对象资源
- 隔离与恢复
- 扩容

集群环境共享与隔离
- Namespace
- Context：过kubectl config set-context命令定义Context，并将Context置于某个命名空间中

Kubernetes中该保障机制的核心如下
- 通过资源限额来确保不同的Pod只能占用指定的资源
- 允许集群的资源被超额分配，以提高集群的资源利用率
- 为Pod划分等级，确保不同等级的Pod有不同的服务质量（QoS），资源不足时，低等级的Pod会被清理，以确保高等级的Pod稳定运行

应用于命名空间的的资源限制：LimitRange资源对象（可以为没有配置资源的Pod提供默认资
源配置）
资源配额管理：ResourceQuota资源对象

通过Metrics Server监控Pod和Node的CPU和内存资源使用数据（相应的ServiceAccount，Deployment，Service）。在Kubernetes新的监控体系中，Metrics Server用于提供核心指标（Core Metrics），包括Node、Pod的CPU和内存使用指标。对其他自定义指标（Custom Metrics）的监控则由Prometheus等组件来完成。

### 日志

在容器中输出到控制台的日志，都会以*-json.log的命名方式保存在/var/lib/docker/containers/目录下，这就为日志采集和后续处理奠定了基础。

## 第十一章 Trouble Shooting指导

查看系统Event(kubectl describe命令)
查看容器日志(kubectl logs <pod_name>命令)
查看Kubernetes服务日志(可以通过使用systemdstatus或journalctl工具来查看系统服务kube-controller-manager等进程的日志)