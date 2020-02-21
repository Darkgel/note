服务治理维护的数据，从根本上说就是“提供者”与“消费者”之间的映射关系

旁路负载：它的特点是并不直接通过代理流量来分发负载流量，而是在RPC客户端中直接埋入负载及其他链路逻辑，这样就省下了网关代理这层，但是由于对外需要统一暴露接口，因此对外的网关仍然需要保留

service mesh是一个“基础设施”层，用于处理服务间通信

集群化 -> 反向代理 -> 旁路负载 -> 服务网格

## 第二章 Service Mesh:以Istio为例

Istio的核心便是“数据平面”与“控制平面”两部分，外加“数据平面”衍生出的“安全控制”

- 数据平面(data plane)：由一系列代理网关构成（Envoy，可以通过API实时接收配置并生效，以此实现动态化的流量代理），包括边界网关（IngressGateway与EngressGateway）及服务之间的Sidecar
- 控制平面(Control Plane): 是对“数据平面”中网关的统一配置与维护平台，这里的配置不仅包含请求路由，还包括限流降级、熔断、日志收集、性能监控等各种请求相关信息。（划分成Pilot，Mixer，Citadel三个小模块）

在默认情况下，Sidecar中并没有任何特别的配置，所有的流量都通过之前的k8s域名走DNS解析走掉了。这样，就会在Pod之间依靠k8s那套做一个自动的负载均衡。

### 数据平面

Envoy的xDS-API（xDS是一类服务发现的统称）

- SDS/EDS(Service/Endpoint Discovery Service): 节点发现服务
- CDS(Cluster Discovery Service): 集群发现服务（集群指Envoy接管的服务集群，例如三台机器组成了一个服务集群提供reviews服务）
- RDS(Route Discovery Service): 路由规则发现服务
- LDS(Listener Discovery Service): 监听器发现服务

istio支持两种形式的错误注入：延时（delays）与丢包(aborts)

### 控制平面

控制平台的三大核心组件：Pilot(配置维护)、Mixer(策略执行)、Citadel(安全相关)

#### Pilot

Pilot的基本工作就是为数据平台的Sidecar提供服务发现能力，让服务之间可以相互发现，并根据配置的策略进行调用。

#### Mixer

Mixer负责在服务调用之间实施控制策略，同时在调用间收集请求的遥测数据（例如调用时长，异常统计等）

实现原理：每个请求在到达Sidecar的时候都需要向Mixer发起一次逻辑请求，以进行前置检查；而在每次请求结束之后，还需要再次向Mixer做一次汇报

对于Mixer来说，三个基础的配置是数据实例(Instance)，规则(Rule)与处理器(Handler)。这三个配置将请求属性与后端的扩展插件结合起来，组成请求策略执行的数据链路。

#### 安全控制

Istio的安全体系是由下列部分组成的。

- Citedel：密钥与证书的管理系统
- Sidecar与边界网关：为客户端与服务端接口提供安全服务
- Pilot：用于向网关分发安全认证策略与安全命名信息(Secure Naming Information)
- Mixer: 用于对链路请求进行鉴权与审计

## 第三章 理解Istio服务网格

### K8s服务组网原理

k8s会为每个Service分配一个ClusterIP，服务之间就是以此来寻址并发起调用的，然而这个ClisterIP并没有绑定任何网卡，实际上它是一个虚拟的标识，是基于软件来转发的，并且在k8s不同的版本中实现方式也不同(iptables/IPVS)

k8s的services本身只是一个虚拟的映射关系，真正提供服务的还是Pod，Sidecar这是被注入此处的。

k8s原生的服务发现依赖的是DNS(kubedns/CoreDNS)，每个k8s环境中都会有一个DNS服务，存在于kube-system这个命名空间下

在没有使用Istio之前，k8s本身是通过pause容器来将Pod中的所有容器整合起来，再通过CNI构建起来的基础网络来联通各服务的，其服务发现使用的是DNS。

### Sidecar流量劫持的原理

pod中应用容器与istio-proxy两个容器的ip地址相同，即两个容器共享了网络。istio在pod启动的时候使用自己的特殊pause容器istio-proxy替换掉了k8s官方的pause，使得应用与istio-proxy处于同一个网络栈下，因此，可以看到应用的监听端口以及istio-proxy容器里envoy的代理端口15001及管理端口15000

Istio实际上使用了proxy_init这个容器来对Pod进行初始化。这个容器运行了一个脚本，通过iptables及iproute2这个工具对Linux网络流量配置了过滤规则来实现Sidecar的流量劫持

### Istio服务组网

服务A到达服务B的关键节点

- istio-proxy容器
- kubelet系统服务
- CNI网络插件

Envoy怎么知道如哪里获取xDS记录？Envoy是动态配置的，但是也需要一些静态配置来作为一切的入口（例如提前静态配置指定xDS服务器）

## 第四章 Istio周边生态一览

常见组件

- 链路跟踪Jaeger/Zipkin
- 分布式监控Prometheus
- 监控大盘Grafana(数据展示平面，更多用于时序相关的展示)
- 分布式日志Fluentd(配合elasticsearch和kibana使用，fluentd是数据收集和处理的地方)
- 服务图谱servicegraph

## 第五章 Istio部分源码剖析

Istio是一个生态，在设计之初便是从平台化的思维入手。
它的设计理念在于，将微服务架构中众多零散的服务节点，通过sidecar这种非侵入方式给串接起来。

### Envoy

Sidecar是通过共享容器网络命名空间，再通过IPVS的方式来实现流量接管的。

### Pilot

pilot本身分为discovery与agent两个模块（pilot的主要工作是将服务注册及路由信息转译至数据平面）

- discovery负责路由及其他配置的管理
- agent负责Envoy(sidecar)生命周期相关的管理，它与envoy是在同一个容器里的

当一个CRD资源发生变化时，Pilot需要负责监听其变化并反映到抽象模型(Abstract Model)中,然后再通过ADS(Aggreated Discovery Service)将配置下发给网格中的Envoy节点

Pilot会向Envoy推送数据，Envoy也会主动向DiscoveryServer请求数据

平台服务注册注册系统 -> Pilot抽象模型 -> Envoy xDS

### Mixer

数据最初是从请求中携带的请求属性(Attribute)来的，Mixer会依据模板的配置，生成对应的实例(Instance)，然后对数据加工处理(例如设置默认值什么的)，再将最后产生的数据转发给适配器。

整个Mixer系统的缓存分为两级：一级是放置在Envoy中，另一级则放置在Mixer服务端

## 第六章 服务网格企业实践

