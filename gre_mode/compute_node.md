## 计算节点
以抽象系统架构的图表为例，Compute节点上包括两台虚拟机VM1和VM2，分别经过一个网桥（如qbr-XXX）连接到br-int网桥上。br-int网桥再经过br-tun网桥（物理网络是GRE实现）连接到物理主机外部网络。

对于物理网络通过vlan来隔离的情况，一般会存在一个br-eth网桥。

### qbr
在VM1中，虚拟机的网卡实际上连接到了物理机的一个TAP设备（即A，常见名称如tap-XXX）上，A则进一步通过VETH pair（A-B）连接到网桥qbr-XXX的端口vnet0（端口B）上，之后再通过VETH pair（C-D）连到br-int网桥上。一般C的名字格式为qvb-XXX，而D的名字格式为qvo-XXX。注意它们的名称除了前缀外，后面的id都是一样的，表示位于同一个虚拟机网络到物理机网络的连接上。

之所以TAP设备A没有直接连接到网桥br-int上，是因为OpenStack需要通过iptables实现security group的安全策略功能。目前openvswitch并不支持应用iptables规则的Tap设备。

因为qbr的存在主要是为了辅助iptables来实现security group功能，有时候也被称为firewall bridge。详见security group部分的分析。

### br-int
一个典型的br-int的端口如下所示：
```
# ovs-vsctl show
Bridge br-int
    Port "qvo-XXX"
        tag: 1
        Interface "qvo-XXX"
    Port patch-tun
        Interface patch-tun
            type: patch
            options: {peer=patch-int}
    Port br-int
        Interface br-int
            type: internal
```
其中br-int为内部端口。

端口patch-tun（即端口E，端口号为1）连接到br-tun上，实现到外部网络的隧道。
端口qvo-XXX（即端口D，端口号为2）带有tag1，说明这个口是一个1号vlan的access端口。虚拟机发出的从该端口到达br-int的网包将被自动带上vlan tag 1，而其他带有vlan tag 1的网包则可以在去掉vlan tag后从该端口发出（具体请查询vlan access端口）。这个vlan tag是用来实现不同网络相互隔离的，比如租户创建一个网络（neutron net-create），则会被分配一个唯一的vlan tag。

br-int在GRE模式中作为一个NORMAL交换机使用，因此有效规则只有一条正常转发。如果两个在同一主机上的vm属于同一个tenant的（同一个vlan tag），则它们之间的通信只需要经过br-int即可。
```
# ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=10727.864s, table=0, n_packets=198, n_bytes=17288, idle_age=13, priority=1 actions=NORMAL
```
### br-tun
一个典型的br-tun上的端口类似：
```
Bridge br-tun
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "gre-1"
            Interface "gre-1"
                type: gre
                options: {in_key=flow, local_ip="10.0.0.101", out_key=flow, remote_ip="10.0.0.100"}
        Port br-tun
            Interface br-tun
                type: internal
```
其中patch-int（即端口F，端口号为1）是连接到br-int上的veth pair的端口，gre-1口（即端口G，端口号为2）对应vm到外面的隧道。

gre-1端口是虚拟gre端口，当网包发送到这个端口的时候，会经过内核封包，然后从10.0.0.101发送到10.0.0.100，即从本地的物理网卡（10.0.0.101）发出。

br-tun将带有vlan tag的vm跟外部通信的流量转换到对应的gre隧道，这上面要实现主要的转换逻辑，规则要复杂，一般通过多张表来实现。

典型的转发规则为：
```
# ovs-ofctl dump-flows br-tun
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=10970.064s, table=0, n_packets=189, n_bytes=16232, idle_age=16, priority=1,in_port=1 actions=resubmit(,1)
 cookie=0x0, duration=10906.954s, table=0, n_packets=29, n_bytes=5736, idle_age=16, priority=1,in_port=2 actions=resubmit(,2)
 cookie=0x0, duration=10969.922s, table=0, n_packets=3, n_bytes=230, idle_age=10962, priority=0 actions=drop
 cookie=0x0, duration=10969.777s, table=1, n_packets=26, n_bytes=5266, idle_age=16, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0x0, duration=10969.631s, table=1, n_packets=163, n_bytes=10966, idle_age=21, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,21)
 cookie=0x0, duration=688.456s, table=2, n_packets=29, n_bytes=5736, idle_age=16, priority=1,tun_id=0x1 actions=mod_vlan_vid:1,resubmit(,10)
 cookie=0x0, duration=10969.488s, table=2, n_packets=0, n_bytes=0, idle_age=10969, priority=0 actions=drop
 cookie=0x0, duration=10969.343s, table=3, n_packets=0, n_bytes=0, idle_age=10969, priority=0 actions=drop
 cookie=0x0, duration=10969.2s, table=10, n_packets=29, n_bytes=5736, idle_age=16, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
 cookie=0x0, duration=682.603s, table=20, n_packets=26, n_bytes=5266, hard_timeout=300, idle_age=16, hard_age=16, priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:32:0d:db actions=load:0->NXM_OF_VLAN_TCI[],load:0x1->NXM_NX_TUN_ID[],output:2
 cookie=0x0, duration=10969.057s, table=20, n_packets=0, n_bytes=0, idle_age=10969, priority=0 actions=resubmit(,21)
 cookie=0x0, duration=688.6s, table=21, n_packets=161, n_bytes=10818, idle_age=21, priority=1,dl_vlan=1 actions=strip_vlan,set_tunnel:0x1,output:2
 cookie=0x0, duration=10968.912s, table=21, n_packets=2, n_bytes=148, idle_age=689, priority=0 actions=drop
```
其中，表0中有3条规则：从端口1（即patch-int）来的，扔到表1，从端口2（即gre-1）来的，扔到表2。
```
cookie=0x0, duration=10970.064s, table=0, n_packets=189, n_bytes=16232, idle_age=16, priority=1,in_port=1 actions=resubmit(,1)
 cookie=0x0, duration=10906.954s, table=0, n_packets=29, n_bytes=5736, idle_age=16, priority=1,in_port=2 actions=resubmit(,2)
 cookie=0x0, duration=10969.922s, table=0, n_packets=3, n_bytes=230, idle_age=10962, priority=0 actions=drop
```
表1有2条规则：如果是单播（00:00:00:00:00:00/01:00:00:00:00:00），则扔到表20；如果是多播等（01:00:00:00:00:00/01:00:00:00:00:00），则扔到表21。
```
cookie=0x0, duration=10969.777s, table=1, n_packets=26, n_bytes=5266, idle_age=16, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)
 cookie=0x0, duration=10969.631s, table=1, n_packets=163, n_bytes=10966, idle_age=21, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,21)
```
表2有2条规则：如果是tunnel 1的网包，则修改其vlan id为1，并扔到表10；非tunnel 1的网包，则丢弃。
```
cookie=0x0, duration=688.456s, table=2, n_packets=29, n_bytes=5736, idle_age=16, priority=1,tun_id=0x1 actions=mod_vlan_vid:1,resubmit(,10)
 cookie=0x0, duration=10969.488s, table=2, n_packets=0, n_bytes=0, idle_age=10969, priority=0 actions=drop
```
表3只有1条规则：丢弃。

表10有一条规则，基于learn行动来创建反向（从gre端口抵达，且目标是到vm的网包）的规则。learn行动并非标准的openflow行动，是openvswitch自身的扩展行动，这个行动可以根据流内容动态来修改流表内容。这条规则首先创建了一条新的流（该流对应vm从br-tun的gre端口发出的规则）：其中table=20表示规则添加在表20；NXM_OF_VLAN_TCI[0..11]表示匹配包自带的vlan id；NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[]表示L2目标地址需要匹配包的L2源地址；load:0->NXM_OF_VLAN_TCI[]，去掉vlan，load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[]，添加tunnel号为原始tunnel号；output:NXM_OF_IN_PORT[]，发出端口为原始包抵达的端口。最后规则将匹配的网包从端口1（即patch-int）发出。
```
cookie=0x0, duration=10969.2s, table=10, n_packets=29, n_bytes=5736, idle_age=16, priority=1 actions=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:1
```
表20中有两条规则，其中第一条即表10中规则利用learn行动创建的流表项，第2条提交其他流到表21。
```
cookie=0x0, duration=682.603s, table=20, n_packets=26, n_bytes=5266, hard_timeout=300, idle_age=16, hard_age=16, priority=1,vlan_tci=0x0001/0x0fff,dl_dst=fa:16:3e:32:0d:db actions=load:0->NXM_OF_VLAN_TCI[],load:0x1->NXM_NX_TUN_ID[],output:2
 cookie=0x0, duration=10969.057s, table=20, n_packets=0, n_bytes=0, idle_age=10969, priority=0 actions=resubmit(,21)
```
表21有2条规则，第一条是匹配所有目标vlan为1的网包，去掉vlan，然后从端口2（gre端口）发出。第二条是丢弃。
```
cookie=0x0, duration=688.6s, table=21, n_packets=161, n_bytes=10818, idle_age=21, priority=1,dl_vlan=1 actions=strip_vlan,set_tunnel:0x1,output:2
 cookie=0x0, duration=10968.912s, table=21, n_packets=2, n_bytes=148, idle_age=689, priority=0 actions=drop
```
这些规则所组成的整体转发逻辑如下图所示。

![Compute节点br-tun的转发逻辑](../_images/compute_br_tun_fwd_logic.png)
