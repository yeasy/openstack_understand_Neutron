# 快速查找安全组规则
从前面分析可以看出，某个vm的安全组相关规则的chain的名字，跟vm的id的前9个字符有关。

因此，要快速查找qbr-XXX上相关的iptables规则，可以用iptables -S列出（默认是filter表）所有链上的规则，其中含有id的链即为虚拟机相关的安全组规则。其中--physdev-in表示即将进入某个网桥的端口，--physdev-out表示即将从某个网桥端口发出。
```sh
#iptables -S |grep tap583c7038-d3
-A neutron-openvswi-FORWARD -m physdev --physdev-out tap583c7038-d3 --physdev-is-bridged -j neutron-openvswi-sg-chain
-A neutron-openvswi-FORWARD -m physdev --physdev-in tap583c7038-d3 --physdev-is-bridged -j neutron-openvswi-sg-chain
-A neutron-openvswi-INPUT -m physdev --physdev-in tap583c7038-d3 --physdev-is-bridged -j neutron-openvswi-o583c7038-d
-A neutron-openvswi-sg-chain -m physdev --physdev-out tap583c7038-d3 --physdev-is-bridged -j neutron-openvswi-i583c7038-d
-A neutron-openvswi-sg-chain -m physdev --physdev-in tap583c7038-d3 --physdev-is-bridged -j neutron-openvswi-o583c7038-d
```
可以看出，进出tap-XXX口的FORWARD链上的流量都被扔到了neutron-openvswi-sg-chain这个链，neutron-openvswi-sg-chain上是security group具体的实现（两条规则，访问虚拟机的流量扔给neutron-openvswi-i583c7038-d；从虚拟机出来的扔给neutron-openvswi-o583c7038-d）。
