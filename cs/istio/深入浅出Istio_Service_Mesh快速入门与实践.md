# 深入浅出Istio

## 第三章 Istio基本介绍

Istio的特性：
- 连接
- 安全
- 策略
- 观察

Istio关键组件
- Pilot: 流量管理
  - 从Kubernetes或者其他平台的注册中心获取服务信息，完成服务发现过程
  - 读取Istio的各项控制配置，在进行转换之后，将其发给数据面进行实施
- Mixer：预检和汇报（分别为调用前和调用后）
- Cidadel：证书管理
- Sidecar(Envoy)：负责控制面对网格控制的实际执行

在Istio的默认实现中，Istio利用istio-init初始化容器中的iptables指令，对所在Pod的流量进行劫持，从而接管Pod中应用的通信过程，如此一来，就获得了通信的控制权，控制面的控制目的最终得以实现。（Kubernetes中，在同一个Pod内的多个容器之间，网络栈是共享的，这正是Sidecar模式的实现基础）

### 核心配置对象

Istio中的资源分为三组进行管理，分别是networking.istio.io、config.istio.io及authentication.istio.io

networking.istio.io：
- VirtualService：是一个控制中心。它的功能简单说来就是：定义一组条件，将符合该条件的流量（从哪来）按照在对象中配置的对应策略进行处理，最后将路由转到匹配的目标中（到哪去）。
  - 主要组成部分：
    - Host：该对象所负责的主机名称（在k8s中可以是服务名）
    - Gateway：流量的来源网关。省略则代表使用的网关名为“mesh”（即默认的网格内部服务互联所用的网关）
    - 路由对象：符合前面的Host和Gateway的条件，就需要根据实际协议对流量的处理方式进行甄别
- Gateway：在访问服务时，不论是网格内部的服务互访，还是通过Ingress进入网格的外部流量，首先要经过的设施都是Gateway。
  - 网络边缘的Ingress流量，会通过对应的Istio IngressGateway Controller进入
  - 网格内部的服务互访，则通过虚拟的mesh网关进行（mesh网关代表网格内部的所有Sidecar）
  - Pilot会根据Gateway和主机名进行检索，如果存在对应的VirtualService，则交由VirtualService处理；如果是Mesh Gateway且不存在对应这一主机名的VirtualService，则尝试调用Kubernetes Service；如果不存在，则发生404错误。
- TCP/TLS/HTTP Route：路由对象目前可以是HTTP、TCP或者TLS中的一个，分别针对不同的协议进行工作
  - 匹配条件
  - 目的路由
- DestinationWeight：DestinationWeight指到某个目标（Destination对象）的流量权重，这就意味着，多个目标可以同时为该VirtualService提供服务，并按照权重进行流量分配
- Destination：目标对象（Destination）由Subset和Port两个元素组成
  - Subset顾名思义，就是指服务的一个子集，它在Kubernetes中代表使用标签选择器区分的不同Pod（例如两个Deployment）

流量路径：Gateway -> VirtalService -> tcp/tls/http Route -> DestinationWeight -> Subset:Port

config.istio.io（用于为Mixer组件提供配置）：
- Rule: Rule对象是Mixer的入口，其中包含一个match成员和一个逻辑表达式，只有符合表达式判断的数据才会被交给Action处理。逻辑表达式中的变量被称为attribute（属性），其中的内容来自Envoy提交的数据。
- Action: 负责解决的问题就是：将符合入口标准的数据，在用什么方式加工之后，交给哪个适配器进行处理
  - Action包含两个成员对象：一个是Instance，使用Template对接收到的数据进行处理；一个是Handler，代表一个适配器的实例，用于接收处理后的数据。
- Instance: Instance主要用于为进入的数据选择一个模板，并在数据中抽取某些字段作为模板的参数，传输给模板进行处理
- Adapter: Envoy传出的数据将会通过这些具体运行的Adapter的处理，得到预检结果，或者输出各种监控、日志及跟踪数据
- Template: Template是一个模板，用于对接收到的数据进行再加工(Template就是这样一种工具，在用户编制模板对象之后，经过模板处理的原始数据会被转换为符合适配器输入要求的数据格式，这样就可以在Instance字段中引用了)
- Handler: Handler对象用于对Adapter进行实例化

Mixer对数据的处理过程：Rule -> Match -> Action -> Instance -> Template -> Handler -> Adapter

authentication.istio.io(定义认证策略)
- Policy: 指定服务一级的认证策略
  - 策略目标: 包含服务名称（或主机名称）及服务端口号
  - 认证方法: 由两个可选部分组成，分别是用于设置服务间认证的peers子对象，以及用于设置终端认证的origins子对象。
- MeshPolicy: MeshPolicy只能被命名为“default”，它代表的是所有网格内部应用的默认认证策略，其余部分内容和Policy一致。

rbac.istio.io(和Kubernetes颇为相似的RBAC访问控制系统)
- ServiceRole
- ServiceRoleBinding

## 第四章 Istio快速入门

init container中的istio-init容器用于初始化流量劫持

Istio的注入要求：没有Service的Deployment是无法被Istio发现并进行操作的。

不论是否进行进一步的流量控制，都建议为网格中的服务创建默认的路由规则，以防发生意料之外的访问结果。

## 第五章 用Helm部署Istio

默认情况下，Pilot会监控所有命名空间的的服务变化。

## 第六章 Istio的常用功能

手动注入sidecar，实际上就是对Deployment对象进行修改，添加相应的内容，主要是sidecar容器和istio-init初始化容器

Istio对工作负载的要求
- 目前支持的工作负载：Job、DaemonSet、ReplicaSet、Pod及Deployment
- 要正确命名服务端口：Service对象中的Port部分必须以“协议名”为前缀，目前支持的协议名包括http、http2、mongo、redis和grpc。Istio会根据这些命名来确定为这些端口提供什么样的服务，不符合命名规范的端口会被当作TCP服务，其功能支持范围会大幅缩小。
- 工作负载的Pod必须有关联的Service：为了满足服务发现的需要，所有Pod都必须有关联的服务（另外，官方建议为Pod模板加入两个标签：app和version，分别标注应用名称和版本。这仅仅是个建议，但是Istio的很多默认策略都会引用这两个标签；如果没有这两个标签，就会引发很多不必要的麻烦。）

不管是手工注入还是自动注入，都可以通过编辑istio-system命名空间中的ConfigMap istio-sidecar-injector，来影响注入的效果
自动注入的评估顺序是：Pod注解→NeverInjectSelector→AlwaysInjectSelector→命名空间策略。

如果想看istio相应pod更详细的日志，可以修改Deployment对象，例如
kubectl -n istio-system edit deployment istio-sidecar-injector
加上容器的运行参数：--log_output_level=default:debug

## 第七章 HTTP流量管理

### DestinationRule

在Istio中，建议为每个网格都设置明确的目标访问规则。在通过Istio流量控制之后，会选择明确的子集，根据该规则或者在子集中规定的流量策略来进行访问，这种规则在Istio中被称为DestinationRule。

host：是一个必要字段，代表Kubernetes中的一个Service资源，或者一个由ServiceEntry（会在7.9节讲解出站流量时介绍）定义的外部服务。为了防止Kubernetes不同命名空间中的服务重名，这里强烈建议使用完全限定名，也就是使用FQDN来赋值。

trafficPolicy：是流量策略（负载均衡等）。在DestinationRule和Subsets两级中都可以定义trafficPolicy，在Subset中设置的级别更高。

subsets：在该字段中使用标签选择器来定义不同的子集。

可以提供的功能
- 负载均衡
- 服务熔断

### VirtualService

在没有配置VirtualService的情况下，默认按照kube-proxy的默认随机行为进行访问

Istio建议为每个服务都创建一个默认路由，在访问某一服务的时候，如果没有特定的路由规则，则使用默认的路由规则来访问指定的子集，以此来确保服务在默认情况下的行为稳定性。

除了可以根据Header的信息来进行路由，还可以根据来源服务进行路由（sourceLabels）

VirtualService可以实现的功能
- 流量的拆分和迁移
- 金丝雀部署
- 根据来源服务进行路由
- 对URI进行重定向
- 通信超时控制
- 故障重试控制
- 故障注入测试
- 流量复制

### Gateway

VirtualService对象，都默认包含gateways字段，如果没有指定，那么其默认值是“mesh”
这里的mesh是Istio内部的虚拟Gateway，代表网格内部的所有Sidecar，换句话说：所有网格内部服务之间的互相通信，都是通过这个网关进行的。

selector实际上是一个标签选择器，用于指定由哪些Gateway Pod来负责这个Gateway对象的运行

hosts字段中指明这个Gateway可能要负责的主机名

入口网关通常承担着流量加密的任务，Ingress Gateway也具备这样的能力

Istio在对应用进行注入的时候，会劫持该应用的所有流量，在默认情况下，网格之内的应用是无法访问网格之外的服务的

Istio提供了以下几种方式用于网格外部通信
- 设置Sidecar的流量劫持范围：根据IP地址来告知Sidecar，哪些外部资源可以放开访问
  - helm设置values.yaml中的proxy.includeIPRanges变量
  - 使用Pod注解traffic.sidecar.istio.io/includeOutboundIPRanges，表明劫持范围
- 注册ServiceEntry：把网格外部的服务使用ServiceEntry的方式注册到网格内部

### Gateway控制器

可以使用helm新建Gateway控制器