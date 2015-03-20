### br-int
集成网桥 br-int 规则比较简单，作为一个正常的二层交换机使用。无论下面虚拟化层是哪种技术实现，集成网桥是看不到的，只知道根据 vlan 和 mac 进行转发。

所连接接口包括：
* tap-xxx，连接到网络 DHCP 服务的命名空间；
* qr-xxx，连接到路由服务的命名空间；
* 往外的 patch-tun 接口，连接到 br-tun 网桥。

其中网络服务接口上会绑定内部 vlan 号，每个号对应一个网络。

```sh
Bridge br-int
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port "qr-694450d6-f6"
            tag: 1
            Interface "qr-694450d6-f6"
                type: internal
        Port "tap13685e28-b0"
            tag: 1
            Interface "tap13685e28-b0"
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
```

转发规则表 0 中是对所有包进行 NORMAL，表 23 中是所有包直接丢弃（是否后面将安全组规则在这里实现？）。

```sh
$ sudo ovs-ofctl dump-flows br-int
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=52889.682s, table=0, n_packets=161, n_bytes=39290, idle_age=13, priority=1 actions=NORMAL
 cookie=0x0, duration=52889.451s, table=23, n_packets=0, n_bytes=0, idle_age=52889, priority=0 actions=drop

```
