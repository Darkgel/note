需要进一步研究
- 不论是网格内部的服务互访，还是通过Ingress进入网格的外部流量，首先要经过的设施都是Gateway
- Pilot会根据Gateway和主机名进行检索，如果存在对应的VirtualService，则交由VirtualService处理；如果是Mesh Gateway且不存在对应这一主机名的VirtualService，则尝试调用Kubernetes Service；如果不存在，则发生404错误。

端口转发(这样就可以通过本地的端口访问到相应的pod)
kubectl -n istio-system port-forward pod-name port:port &

DestinationRule中的host使用完全限定名（VirtualService同样）