# 防火墙即服务

熟悉防火墙的都知道，防火墙一般放在网关上，用来隔离子网之间的访问。因此，防火墙即服务（FireWall as a Service）也是在网络节点上（具体说来是在路由器命名空间中）来实现。

目前，OpenStack 中实现防火墙还是基于 Linux 系统自带的 iptables，所以大家对于其性能和功能就不要抱太大的期望了。

一个可能混淆的概念是安全组（Security Group），安全组的对象是主机，由L2 Agent来实现，比如neutron_openvswitch_agent 和 neutron_linuxbridge_agent，会在计算节点上通过配置 iptables 规则来限制虚拟机的进出访问。防火墙可以在安全组之前隔离外部过来的恶意流量，但是对于虚机出来的流量则无法过滤（除非它要跨子网）。


可以同时部署防火墙和安全组实现双重防护。
