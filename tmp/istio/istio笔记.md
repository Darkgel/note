# istio

Like other Istio configuration, the API(istio的api) is specified using Kubernetes custom resource definitions (CRDs), which you can configure using YAML, as you’ll see in the examples.

## Traffic Management

traffic management API resources:

- Virtual services ：defines the rules that control how requests for a service are routed within an Istio service mesh.（服务调用者定义）
- Destination rules ：configures the set of policies to be applied to a request after VirtualService routing has occurred.（服务提供者定义）
- Gateways ：configures a load balancer operating at the edge of the mesh for HTTP/TCP ingress traffic to a mesh application or egress traffic to external services.
- Service entries ：is commonly used to enable requests to services outside of an Istio service mesh.（通常用于定义集群外服务）
- Sidecars ：configures one or more sidecar proxies attached to application workloads running inside the mesh.

## Policies

- Rate limiting to dynamically limit the traffic to a service
- Denials, whitelists, and blacklists, to restrict access to services
- Header rewrites and redirects
