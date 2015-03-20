# 网络名字空间
在 Linux 中，网络名字空间可以被认为是隔离的拥有单独网络栈（网卡、路由转发表、iptables）的环境。网络名字空间经常用来隔离网络设备和服务，只有拥有同样网络名字空间的设备，才能看到彼此。

可以用ip netns list命令来查看已经存在的名字空间。
```sh
$ ip net
qdhcp-ea3928dc-b1fd-4a1a-940e-82b8c55214e6
qrouter-40fff075-d3a2-477b-942c-6b1adb42e35e
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



