### br-ex
核心接口有两个。

一个是挂载的物理接口上，如 eth0，网包将从这个接口发送到外部网络上。

另外一个是 qg-xxx 这样的接口，是连接到 router 服务的网络名字空间中，里面绑定一个路由器的外部 IP，作为 nAT 时候的地址，另外，网络中的 floating IP 也放在这个网络名字空间中。

```sh
Bridge br-ex
        Port "eth0"
            Interface "eth0"
        Port br-ex
            Interface br-ex
                type: internal
        Port "qg-e76de35e-90"
            Interface "qg-e76de35e-90"
                type: internal

```

网桥的规则也很简单，作为一个正常的二层转发设备即可。
```
$ sudo ovs-ofctl dump-flows br-ex
NXST_FLOW reply (xid=0x4):
 cookie=0x0, duration=75072.257s, table=0, n_packets=352212, n_bytes=85641148, idle_age=0, hard_age=65534, priority=0 actions=NORMAL
```
