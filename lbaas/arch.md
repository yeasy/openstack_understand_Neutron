## 实现细节
跟大部分的高级服务一样，LBaaS 在网络节点上实现。

### qlbaas 命名空间
在启动了 LBaaS 之后，网络节点上会多一个 qlbaas-xxx 命名空间，其中一个 tap 类型端口，绑定了我们定义的 VIP。
```sh
$ sudo ip netns exec qlbaas-574a31da-a28a-449f-8c9d-3d3687c3c02a ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
52: tapfb358342-59: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN group default
    link/ether fa:16:3e:68:77:37 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.250/24 brd 10.0.0.255 scope global tapfb358342-59
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fe68:7737/64 scope link
       valid_lft forever preferred_lft forever
```

该 tap 类型端口，会连接到网络节点的 br-int 网桥上。


### HAProxy
此时在网络节点上查看进程，会发现一个 haproxy 进程。
```sh
$ ps aux|grep haproxy
nobody   17331  0.0  0.0  20376  1104 ?        Ss   15:08   0:00 haproxy -f /opt/stack/data/neutron/lbaas/574a31da-a28a-449f-8c9d-3d3687c3c02a/conf -p /opt/stack/data/neutron/lbaas/574a31da-a28a-449f-8c9d-3d3687c3c02a/pid -sf 17323
```

查看 HAProxy 的配置文件
```sh
$ cat /opt/stack/data/neutron/lbaas/574a31da-a28a-449f-8c9d-3d3687c3c02a/conf
global
        daemon
        user nobody
        group nogroup
        log /dev/log local0
        log /dev/log local1 notice
        stats socket /opt/stack/data/neutron/lbaas/574a31da-a28a-449f-8c9d-3d3687c3c02a/sock mode 0666 level user
defaults
        log global
        retries 3
        option redispatch
        timeout connect 5000
        timeout client 50000
        timeout server 50000
frontend df25900f-53b1-4aab-8436-8411ec710445
        option tcplog
        bind 10.0.0.250:22
        mode tcp
        default_backend 574a31da-a28a-449f-8c9d-3d3687c3c02a
backend 574a31da-a28a-449f-8c9d-3d3687c3c02a
        mode tcp
        balance leastconn
        server 45481748-3104-462c-ae4b-fae0f4b08c20 10.0.0.2:22 weight 1
        server 227b69fe-8685-4df6-b2a5-2f964c74886c 10.0.0.4:22 weight 1
```

可见，里面的配置信息，跟我们之前配置的完全一致。

当有请求访问 10.0.0.250:22 的时候，最终会到达该 qlbaas 命名空间，请求被 HAProxy 收到，按照负载均衡策略，转发给合适的虚机。

可以通过 ssh 进行测试。
```sh
$ sudo ip netns exec qrouter-40fff075-d3a2-477b-942c-6b1adb42e35e ssh 10.0.0.250
```
在后端的虚机抓包，SSH 请求是 10.0.0.250 -> 10.0.0.2，说明实际上是 HAProxy 转发了请求。
