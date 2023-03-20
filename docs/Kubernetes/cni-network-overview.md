# CNI网络概述

CNI:容器网络接口

CNI解决的是夸主机网络通信

条件:

- 一个pod分配唯一ip
- node可以访问任何节点pod
- pod可以访问所有pod

网络插件实现模式: 路由,隧道

flannel: vxlan(隧道), host-gw(路由), udp(性能太差)
calico: ipip(隧道), bgp(路由)

路由方案对现有网络是有要求的(比如写路由表),性能好,直接路由转发,数据包无需额外再封装,一般要求二层可达,大二层网络
隧道方案只要三层可达就基本能通信

CNI网络选型考虑:

1. 集群规模(小规模100台 flannel host-gw)
2. 是否需要网络策略
3. 隧道 | 路由 现有网络是否有限制,比如主机写路由表,BGP是否可以通信
4. 维护成本,calico比较复杂,维护成本高