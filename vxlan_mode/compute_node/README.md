## 计算节点

主要包括两个网桥：集成网桥 br-int 和 隧道网桥 br-tun。

```sh
$ sudo ovs-vsctl show
225f3eb5-6059-4063-99c3-8666915c9c55
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
    Bridge br-tun
        fail_mode: secure
        Port "vxlan-0a00644d"
            Interface "vxlan-0a00644d"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="10.0.100.88", out_key=flow, remote_ip="10.0.100.77"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port br-tun
            Interface br-tun
                type: internal
    ovs_version: "2.0.2"
```

安全网桥可以通过 `brctl show` 命令看到，该网桥主要用于绑定控制组的 iptables 规则，跟转发无直接关系。
```sh
~$ brctl show
bridge name     bridge id               STP enabled     interfaces
qbrf47c62b0-db          8000.56a7904c418d       no              qvbf47c62b0-db
                                                        tapf47c62b0-db
```
