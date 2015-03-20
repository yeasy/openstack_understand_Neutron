## 网络节点

网络节点担负着进行网络服务的任务，包括DHCP、路由和高级网络服务等。一般包括三个网桥：br-tun、br-int 和 br-ex。

```sh
$ sudo ovs-vsctl show
49761e8e-031f-4a60-b838-28bb82aac7b7
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
    Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
        Port "qg-e76de35e-90"
            Interface "qg-e76de35e-90"
                type: internal
    Bridge br-tun
        fail_mode: secure
        Port br-tun
            Interface br-tun
                type: internal
        Port "vxlan-0a006458"
            Interface "vxlan-0a006458"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="10.0.100.77", out_key=flow, remote_ip="10.0.100.88"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
    ovs_version: "2.0.2"
```
