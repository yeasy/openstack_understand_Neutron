### br-int
集成网桥 br-int 规则比较简单，作为一个正常的二层交换机使用。无论下面虚拟化层是哪种技术实现，集成网桥是看不到的，只知道根据 vlan 和 mac 进行转发。

所连接接口除了从安全网桥过来的 qvo-xxx（每个虚拟机会有一个），就是一个往外的 patch-tun 接口，连接到 br-tun 网桥。

其中，qvo-xxx 接口上会为每个网络分配一个内部 vlan 号，比如这里是同一个网络启动了两台虚机，所以 tag 都为 1。

```sh
 Bridge br-int
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port "qvoc4493802-43"
            tag: 1
            Interface "qvoc4493802-43"
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "qvof47c62b0-db"
            tag: 1
            Interface "qvof47c62b0-db"
```

转发规则表 0 中是对所有包进行 NORMAL，表 23 中是所有包直接丢弃（是否后面将安全组规则在这里实现？）。

```sh
$ sudo ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=52889.682s, table=0, n_packets=161, n_bytes=39290, idle_age=13, priority=1 actions=NORMAL
 cookie=0x0, duration=52889.451s, table=23, n_packets=0, n_bytes=0, idle_age=52889, priority=0 actions=drop

```
