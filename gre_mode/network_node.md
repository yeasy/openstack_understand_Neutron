## 网络节点
### br-tun
```
Bridge br-tun
        Port br-tun
            Interface br-tun
                type: internal
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "gre-2"
            Interface "gre-2"
                type: gre
                options: {in_key=flow, local_ip="10.0.0.100", out_key=flow, remote_ip="10.0.0.101"}
```
Compute节点上发往GRE隧道的网包最终抵达Network节点上的br-tun，该网桥的规则包括：
```
# ovs-ofctl dump-flows br-tun
NXST_FLOW reply (xid=0x4):
cookie=0x0, duration=19596.862s, table=0, n_packets=344, n_bytes=66762, idle_age=4, priority=1,in_port=1 actions=resubmit(,1)
 cookie=0x0, duration=19537.588s, table=0, n_packets=625, n_bytes=125972, idle_age=4, priority=1,in_port=2 actions=resubmit(,2)
 cookie=0x0, duration=19596.602s, table=0, n_packets=2, n_bytes=140, idle_age=19590, priority=0 actions=drop
 cookie=0x0, duration=19596.343s, table=1, n_packets=323, n_bytes=65252, idle_age=4, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0x0, duration=19596.082s, table=1, n_packets=21, n_bytes=1510, idle_age=5027, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,21)
 cookie=0x0, duration=9356.289s, table=2, n_packets=625, n_bytes=125972, idle_age=4, priority=1,tun_id=0x1 actions=mod_vlan_vid:1,resubmit(,10)
 cookie=0x0, duration=19595.821s, table=2, n_packets=0, n_bytes=0, idle_age=19595, priority=0 actions=drop
 cookie=0x0, duration=19595.554s, table=3, n_packets=0, n_bytes=0, idle_age=19595, priority=0 actions=drop
 cookie=0x0, duration=19595.292s, table=10, n_packets=625, n_bytes=125972, idle_age=4, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
 cookie=0x0, duration=9314.338s, table=20, n_packets=323, n_bytes=65252, hard_timeout=300, idle_age=4, hard_age=3, priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:cb:11:f6 actions=load:0->NXM_OF_VLAN_TCI[],load:0x1->NXM_NX_TUN_ID[],output:2
 cookie=0x0, duration=19595.026s, table=20, n_packets=0, n_bytes=0, idle_age=19595, priority=0 actions=resubmit(,21)
 cookie=0x0, duration=9356.592s, table=21, n_packets=9, n_bytes=586, idle_age=5027, priority=1,dl_vlan=1 actions=strip_vlan,set_tunnel:0x1,output:2
 cookie=0x0, duration=19594.759s, table=21, n_packets=12, n_bytes=924, idle_age=5057, priority=0 actions=drop
```
这些规则跟Compute节点上br-tun的规则相似，完成tunnel跟vlan之间的转换。

### br-int
```
Bridge br-int
        Port "qr-ff19a58b-3d"
            tag: 1
            Interface "qr-ff19a58b-3d"
                type: internal
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "tap4385f950-8b"
            tag: 1
            Interface "tap4385f950-8b"
                type: internal
```
该集成网桥上挂载了很多进程来提供网络服务，包括路由器、DHCP服务器等。这些进程不同的租户可能都需要，彼此的地址空间可能冲突，也可能跟物理网络的地址空间冲突，因此都运行在独立的网络名字空间中。
规则跟computer节点的br-int规则一致，表现为一个正常交换机。
```sh
# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=18198.244s, table=0, n_packets=849, n_bytes=164654, idle_age=43, priority=1 actions=NORMAL
```
### 网络名字空间
在linux中，网络名字空间可以被认为是隔离的拥有单独网络栈（网卡、路由转发表、iptables）的环境。网络名字空间经常用来隔离网络设备和服务，只有拥有同样网络名字空间的设备，才能看到彼此。
可以用ip netns list命令来查看已经存在的名字空间。
```sh
# ip netns
qdhcp-88b1609c-68e0-49ca-a658-f1edff54a264
qrouter-2d214fde-293c-4d64-8062-797f80ae2d8f
```
qdhcp开头的名字空间是dhcp服务器使用的，qrouter开头的则是router服务使用的。
可以通过 `ip netns exec namespaceid command` 来在指定的网络名字空间中执行网络命令，例如
```sh
# ip netns exec qdhcp-88b1609c-68e0-49ca-a658-f1edff54a264 ip addr
71: ns-f14c598d-98: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:10:2f:03 brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.3/24 brd 10.1.0.255 scope global ns-f14c598d-98
    inet6 fe80::f816:3eff:fe10:2f03/64 scope link
       valid_lft forever preferred_lft forever
```
可以看到，dhcp服务的网络名字空间中只有一个网络接口“ns-f14c598d-98”，它连接到br-int的tapf14c598d-98接口上。

### dhcp 服务
dhcp服务是通过dnsmasq进程（轻量级服务器，可以提供dns、dhcp、tftp等服务）来实现的，该进程绑定到dhcp名字空间中的br-int的接口上。可以查看相关的进程。
```sh
# ps -fe | grep 88b1609c-68e0-49ca-a658-f1edff54a264
nobody   23195     1  0 Oct26 ?        00:00:00 dnsmasq --no-hosts --no-resolv --strict-order --bind-interfaces --interface=ns-f14c598d-98 --except-interface=lo --pid-file=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/host --dhcp-optsfile=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/opts --dhcp-script=/usr/bin/neutron-dhcp-agent-dnsmasq-lease-update --leasefile-ro --dhcp-range=tag0,10.1.0.0,static,120s --conf-file= --domain=openstacklocal
root     23196 23195  0 Oct26 ?        00:00:00 dnsmasq --no-hosts --no-resolv --strict-order --bind-interfaces --interface=ns-f14c598d-98 --except-interface=lo --pid-file=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/host --dhcp-optsfile=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/opts --dhcp-script=/usr/bin/neutron-dhcp-agent-dnsmasq-lease-update --leasefile-ro --dhcp-range=tag0,10.1.0.0,static,120s --conf-file= --domain=openstacklocal
```


###	router服务
首先，要理解什么是router，router是提供跨subnet的互联功能的。比如用户的内部网络中主机想要访问外部互联网的地址，就需要router来转发（因此，所有跟外部网络的流量都必须经过router）。目前router的实现是通过iptables进行的。
同样的，router服务也运行在自己的名字空间中，可以通过如下命令查看：
```sh
# ip netns exec qrouter-2d214fde-293c-4d64-8062-797f80ae2d8f ip addr
66: qg-d48b49e0-aa: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:5c:a2:ac brd ff:ff:ff:ff:ff:ff
    inet 172.24.4.227/28 brd 172.24.4.239 scope global qg-d48b49e0-aa
    inet 172.24.4.228/32 brd 172.24.4.228 scope global qg-d48b49e0-aa
    inet6 fe80::f816:3eff:fe5c:a2ac/64 scope link
       valid_lft forever preferred_lft forever
68: qr-c2d7dd02-56: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:ea:64:6e brd ff:ff:ff:ff:ff:ff
    inet 10.1.0.1/24 brd 10.1.0.255 scope global qr-c2d7dd02-56
    inet6 fe80::f816:3eff:feea:646e/64 scope link
       valid_lft forever preferred_lft forever
```
可以看出，该名字空间中包括两个网络接口。

第一个接口qg-d48b49e0-aa（即K）是外部接口（qg=q gateway），将路由器的网关指向默认网关（通过router-gateway-set命令指定），这个接口连接到br-ex上的tapd48b49e0-aa（即L）。

第二个接口qr-c2d7dd02-56（即N，qr=q bridge）跟br-int上的tapc2d7dd02-56口（即M）相连，将router进程连接到集成网桥上。

查看该名字空间中的路由表：
```sh
# ip netns exec qrouter-2d214fde-293c-4d64-8062-797f80ae2d8f ip route
172.24.4.224/28 dev qg-d48b49e0-aa  proto kernel  scope link  src 172.24.4.227
10.1.0.0/24 dev qr-c2d7dd02-56  proto kernel  scope link  src 10.1.0.1
default via 172.24.4.225 dev qg-d48b49e0-aa
```
其中，第一条规则是将到172.24.4.224/28段的访问都从网卡qg-d48b49e0-aa（即K）发出。

第二条规则是将到10.1.0.0/24段的访问都从网卡qr-c2d7dd02-56（即N）发出。
最后一条是默认路由，所有的通过qg-d48b49e0-aa网卡（即K）发出。
floating ip服务同样在路由器名字空间中实现，例如如果绑定了外部的floating ip 172.24.4.228到某个虚拟机10.1.0.2，则nat表中规则为：
```sh
# ip netns exec qrouter-2d214fde-293c-4d64-8062-797f80ae2d8f iptables -t nat -S
-P PREROUTING ACCEPT
-P POSTROUTING ACCEPT
-P OUTPUT ACCEPT
-N neutron-l3-agent-OUTPUT
-N neutron-l3-agent-POSTROUTING
-N neutron-l3-agent-PREROUTING
-N neutron-l3-agent-float-snat
-N neutron-l3-agent-snat
-N neutron-postrouting-bottom
-A PREROUTING -j neutron-l3-agent-PREROUTING
-A POSTROUTING -j neutron-l3-agent-POSTROUTING
-A POSTROUTING -j neutron-postrouting-bottom
-A OUTPUT -j neutron-l3-agent-OUTPUT
-A neutron-l3-agent-OUTPUT -d 172.24.4.228/32 -j DNAT --to-destination 10.1.0.2
-A neutron-l3-agent-POSTROUTING ! -i qg-d48b49e0-aa ! -o qg-d48b49e0-aa -m conntrack ! --ctstate DNAT -j ACCEPT
-A neutron-l3-agent-PREROUTING -d 169.254.169.254/32 -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 9697
-A neutron-l3-agent-PREROUTING -d 172.24.4.228/32 -j DNAT --to-destination 10.1.0.2
-A neutron-l3-agent-float-snat -s 10.1.0.2/32 -j SNAT --to-source 172.24.4.228
-A neutron-l3-agent-snat -j neutron-l3-agent-float-snat
-A neutron-l3-agent-snat -s 10.1.0.0/24 -j SNAT --to-source 172.24.4.227
-A neutron-postrouting-bottom -j neutron-l3-agent-snat
```
其中SNAT和DNAT规则完成外部floating ip到内部ip的映射：
```sh
-A neutron-l3-agent-OUTPUT -d 172.24.4.228/32 -j DNAT --to-destination 10.1.0.2
-A neutron-l3-agent-PREROUTING -d 172.24.4.228/32 -j DNAT --to-destination 10.1.0.2
-A neutron-l3-agent-float-snat -s 10.1.0.2/32 -j SNAT --to-source 172.24.4.228
```
另外有一条SNAT规则把所有其他的内部IP出来的流量都映射到外部IP 172.24.4.227。这样即使在内部虚拟机没有外部IP的情况下，也可以发起对外网的访问。
```
-A neutron-l3-agent-snat -s 10.1.0.0/24 -j SNAT --to-source 172.24.4.227
```

### br-ex
```sh
Bridge br-ex
        Port "eth1"
            Interface "eth1"
        Port br-ex
            Interface br-ex
                type: internal
        Port "qg-1c3627de-1b"
            Interface "qg-1c3627de-1b"
                type: internal
```
br-ex上直接连接到外部物理网络，一般情况下网关在物理网络中已经存在，则直接转发即可。
```sh
# ovs-ofctl dump-flows br-ex
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=23431.091s, table=0, n_packets=893539, n_bytes=504805376, idle_age=0, priority=0 actions=NORMAL
```
如果对外部网络的网关地址配置到了br-ex（即br-ex作为一个网关）：
```sh
# ip addr add 172.24.4.225/28 dev br-ex
```
需要将内部虚拟机发出的流量进行SNAT，之后发出。
```sh
# iptables -A FORWARD -d 172.24.4.224/28 -j ACCEPT
# iptables -A FORWARD -s 172.24.4.224/28 -j ACCEPT
# iptables -t nat -I POSTROUTING 1 -s 172.24.4.224/28 -j MASQUERADE
```
