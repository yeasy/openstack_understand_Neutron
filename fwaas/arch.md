## 实现细节

防火墙由 L3 Agent 通过修改 iptables 规则来实现，具体规则在网络节点的路由器命名空间中，作用到该租户所有路由器的 qr-xxx 接口上。

在网络节点上查看其中的 iptables 规则，会发现多了两个 iptables 链，分别处理进出两个方向的流量，被  neutron-vpn-agen-FORWARD  引用。
```sh
$ sudo ip netns exec qrouter-40fff075-d3a2-477b-942c-6b1adb42e35e iptables -nvL
Chain neutron-vpn-agen-iv4d8ed9898 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            state INVALID
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            10.0.0.0/24          tcp spts:1:65535 dpt:22

Chain neutron-vpn-agen-ov4d8ed9898 (1 references)
 pkts bytes target     prot opt in     out     source               destination
    0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0            state INVALID
    0     0 ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            state RELATED,ESTABLISHED
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            10.0.0.0/24          tcp spts:1:65535 dpt:22

```
