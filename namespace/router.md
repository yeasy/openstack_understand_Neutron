## 路由服务
首先，要理解什么是 router，router是提供跨 subnet 的互联功能的。比如用户的内部网络中主机想要访问外部互联网的地址，就需要router来转发（因此，所有跟外部网络的流量都必须经过router）。目前router的实现是通过iptables进行的。

同样的，router服务也运行在自己的名字空间中，可以通过如下命令查看：
```sh
$ sudo ip net exec qrouter-40fff075-d3a2-477b-942c-6b1adb42e35e ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
49: qr-694450d6-f6: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether fa:16:3e:5d:18:10 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.1/24 brd 10.0.0.255 scope global qr-694450d6-f6
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe5d:1810/64 scope link
       valid_lft forever preferred_lft forever
50: qg-e76de35e-90: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether fa:16:3e:70:24:92 brd ff:ff:ff:ff:ff:ff
    inet 9.186.100.2/24 brd 9.186.100.255 scope global qg-e76de35e-90
       valid_lft forever preferred_lft forever
    inet 9.186.100.129/32 brd 9.186.100.129 scope global qg-e76de35e-90
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe70:2492/64 scope link
       valid_lft forever preferred_lft forever
```
可以看出，该名字空间中包括两个网络接口。

第一个接口 qr-694450d6-f6（10.0.0.1）跟 br-int 上的接口相连。即任何从 br-int 来的找 10.0.0.1 （租户的私有网段）的网包都会到达这个接口。

第一个接口 qg-e76de35e-90 连接到 br-ex 上的接口，即任何从外部来的网包，询问 9.186.100.2（默认的静态 NAT 外部地址）或 9.186.100.129（租户申请的 floating IP 地址），都会到达这个接口。

查看该名字空间中的路由表：
```sh
$ sudo ip net exec qrouter-40fff075-d3a2-477b-942c-6b1adb42e35e ip route
default via 9.186.100.1 dev qg-e76de35e-90
9.186.100.0/24 dev qg-e76de35e-90  proto kernel  scope link  src 9.186.100.2
10.0.0.0/24 dev qr-694450d6-f6  proto kernel  scope link  src 10.0.0.1
```
默认情况，以及访问外部网络的时候，休会从 qg-xxx 接口发出，经过 br-ex 发布到外网。

访问租户内网的时候，会从 qr-xxx 接口发出，发给 br-int。

```sh
$ sudo ip net exec qrouter-40fff075-d3a2-477b-942c-6b1adb42e35e iptables -t nat -S
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT
-N neutron-postrouting-bottom
-N neutron-vpn-agen-OUTPUT
-N neutron-vpn-agen-POSTROUTING
-N neutron-vpn-agen-PREROUTING
-N neutron-vpn-agen-float-snat
-N neutron-vpn-agen-snat
-A PREROUTING -j neutron-vpn-agen-PREROUTING
-A OUTPUT -j neutron-vpn-agen-OUTPUT
-A POSTROUTING -j neutron-vpn-agen-POSTROUTING
-A POSTROUTING -j neutron-postrouting-bottom
-A neutron-postrouting-bottom -j neutron-vpn-agen-snat
-A neutron-vpn-agen-OUTPUT -d 9.186.100.129/32 -j DNAT --to-destination 10.0.0.2
-A neutron-vpn-agen-POSTROUTING ! -i qg-e76de35e-90 ! -o qg-e76de35e-90 -m conntrack ! --ctstate DNAT -j ACCEPT
-A neutron-vpn-agen-PREROUTING -d 169.254.169.254/32 -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 9697
-A neutron-vpn-agen-PREROUTING -d 9.186.100.129/32 -j DNAT --to-destination 10.0.0.2
-A neutron-vpn-agen-float-snat -s 10.0.0.2/32 -j SNAT --to-source 9.186.100.129
-A neutron-vpn-agen-snat -j neutron-vpn-agen-float-snat
-A neutron-vpn-agen-snat -s 10.0.0.0/24 -j SNAT --to-source 9.186.100.2
```
其中SNAT和DNAT规则完成外部 floating ip （9.186.100.129）到内部 ip（10.0.0.2） 的映射：
```sh
-A neutron-vpn-agen-OUTPUT -d 9.186.100.129/32 -j DNAT --to-destination 10.0.0.2
-A neutron-vpn-agen-PREROUTING -d 9.186.100.129/32 -j DNAT --to-destination 10.0.0.2
-A neutron-vpn-agen-float-snat -s 10.0.0.2/32 -j SNAT --to-source 9.186.100.129
```

另外有一条SNAT规则把所有其他的内部IP出来的流量都映射到外部IP 9.186.100.2。这样即使在内部虚拟机没有外部IP的情况下，也可以发起对外网的访问。
```
-A neutron-vpn-agen-snat -s 10.0.0.0/24 -j SNAT --to-source 9.186.100.2
```
