## easyOVS
[easyOVS](https://github.com/yeasy/easyOVS) 是一个开源的 OpenvSwitch 虚拟交换机管理工具。使用它，用户可以很轻松的对 OpenvSwitch 的网桥、端口等进行查看，同时它深度整合了 OpenStack （支持 Havana 版本到 Juno 版本） 中网络相关的信息，也是一个十分强大的 Neutron 中各个组件的监测工具。


### 主要功能一览
* 支持 OpenvSwitch 版本 1.4.6 ~ 2.0.2，OpenStack Havana 到 Juno 版本；
* 支持操作系统环境报 Ubuntu、Debian、CentOS、Fedora 和 Redhat；
* 输出结果经过处理，支持彩色输出，十分简洁易读；
* 开启 OpenStack 支持，可以获取端口的地址、mac、vlan 甚至虚拟机关联的 iptables 规则等信息；
* 对流表操作语法更加简洁，并支持通过 id 进行删除；
* 支持 tab 自动补全；
* 支持通过 `-m 'cmd'` 来直接运行命令，无需进入 CLI 操作。

### 安装
安装十分简单，一行代码搞定。
```sh
git clone https://github.com/yeasy/easyOVS.git && sudo bash ./easyOVS/util/install.sh
```

安装成功后，可以使用
```sh
sudo easyovs
```
进入操作界面。

### 打开 OpenStack 支持
由于 OpenStack 组件信息获取需要有相关的认证信息，因此需要在环境变量或者配置文件中进行指定。

#### 环境变量
可以在用户目录的 `.bashrc` 文件中加入
```sh
export OS_USERNAME=demo
export OS_TENANT_NAME=demo
export OS_PASSWORD=admin
export OS_AUTH_URL=http://127.0.0.1:5000/v2.0/
```
#### 配置文件
默认的配置文件在 `/etc/easyovs.conf`，替换为相应的认证信息即可。
```sh
[OS]
auth_url = http://127.0.0.1:5000/v2.0
username = demo
password = admin
tenant_name = demo
```

### 操作命令
#### help
显示帮助信息.

#### list
列出本地的 OpenvSwitch 网桥，例如
```sh
 EasyOVS> list
s1
 Port:		s1-eth2 s1 s1-eth1
 Interface:	s1-eth2 s1 s1-eth1
 Controller:ptcp:6634 tcp:127.0.0.1:6633
 Fail_Mode:	secure
s2
 Port:		s2 s2-eth3 s2-eth2 s2-eth1
 Interface:	s2 s2-eth3 s2-eth2 s2-eth1
 Controller:tcp:127.0.0.1:6633 ptcp:6635
 Fail_Mode:	secure
s3
 Port:		s3-eth1 s3-eth3 s3-eth2 s3
 Interface:	s3-eth1 s3-eth3 s3-eth2 s3
 Controller:ptcp:6636 tcp:127.0.0.1:6633
 Fail_Mode:	secure
```

#### show
`EasyOVS> [bridge|default] show`

显示某个网桥上的端口信息，例如
```sh
 EasyOVS> br-int show
br-int
Intf                Port        Vlan    Type        vmIP            vmMAC
int-br-eth0         15
qvo260209fa-72      11          1                   192.168.0.4     fa:16:3e:0f:17:04
qvo583c7038-d3      2           1                   192.168.0.2     fa:16:3e:9c:dc:3a
qvo8bf9cba2-3f      9           1                   192.168.0.5     fa:16:3e:a2:2f:0e
qvod4de9fe0-6d      8           2                   10.0.0.2        fa:16:3e:38:2b:2e
br-int              LOCAL               internal
```

#### dump
`EasyOVS> [bridge|default] dump`

显示网桥上绑定的流表规则，例如

```sh
EasyOVS> s1 dump
ID TAB PKT       PRI   MATCH                                                       ACT
0  0   0         2400  dl_dst=ff:ff:ff:ff:ff:ff                                    CONTROLLER:65535
1  0   0         2400  arp                                                         CONTROLLER:65535
2  0   0         2400  dl_type=0x88cc                                              CONTROLLER:65535
3  0   0         2400  ip,nw_proto=2                                               CONTROLLER:65535
4  0   0         801   ip                                                          CONTROLLER:65535
5  0   2         800
```

#### addflow
`EasyOVS> [bridge|default] addflow [match] actions=[action]`

添加一条流到网桥，例如

`EasyOVS> br-int addflow priority=3 ip actions=OUTPUT:1`

#### delflow
`EasyOVS> [bridge|default] delflow id1 id2...`

从网桥删除流，其中 id 信息可以从 `dump` 的结果中拿到.


#### set
`EasyOVS> bridge set`

指定默认网桥，同时进入网桥操作模式，指定后进行操作可以忽略网桥信息。
```sh
EasyOVS> set br-int
Set the default bridge to br-int.
EasyOVS: br-int>
```

#### exit
`EasyOVS> exit`

退出网桥模式，或者退出程序.

#### get
`EasyOVS> get`

在网桥模式下，获取当前的网桥名称.
```sh
EasyOVS: br-int> get
Current default bridge is br-int
```

#### ipt
`EasyOVS> ipt vm_ip1, vm_ip2...`

给定虚拟机 IP 地址，显示与它相关的 iptables 规则。需要启用 OpenStack 支持。
```sh
EasyOVS> ipt 192.168.0.2 192.168.0.4
## IP = 192.168.0.2, port = qvo583c7038-d ##
    PKTS	SOURCE          DESTINATION     PROT  OTHER
#IN:
     672	all             all             all   state RELATED,ESTABLISHED
       0	all             all             tcp   tcp dpt:22
       0	all             all             icmp
       0	192.168.0.4     all             all
       3	192.168.0.5     all             all
       8	10.0.0.2        all             all
   85784	192.168.0.3     all             udp   udp spt:67 dpt:68
#OUT:
    196K	all             all             udp   udp spt:68 dpt:67
   86155	all             all             all   state RELATED,ESTABLISHED
    1241	all             all             all
#SRC_FILTER:
   59163	192.168.0.2     all             all   MAC FA:16:3E:9C:DC:3A
## IP = 192.168.0.4, port = qvo260209fa-7 ##
    PKTS	SOURCE          DESTINATION     PROT  OTHER
#IN:
      73	all             all             all   state RELATED,ESTABLISHED
       0	all             all             tcp   tcp dpt:22
       0	all             all             icmp
       0	192.168.0.2     all             all
       0	192.168.0.5     all             all
       0	10.0.0.2        all             all
   11331	192.168.0.3     all             udp   udp spt:67 dpt:68
#OUT:
   30034	all             all             udp   udp spt:68 dpt:67
   11377	all             all             all   state RELATED,ESTABLISHED
      12	all             all             all
#SRC_FILTER:
    9859	192.168.0.4     all             all   MAC FA:16:3E:0F:17:04

```

#### query

`EasyOVS> query port_ip, port_id...`

给定某个的端口的 IP 地址，或者部分端口 id 信息，显示该端口相关的完整信息。需要启用 OpenStack 支持。

```sh
EasyOVS> query 10.0.0.2,c4493802
## port_id = f47c62b0-dbd2-4faa-9cdd-8abc886ce08f
status: ACTIVE
name:
allowed_address_pairs: []
admin_state_up: True
network_id: ea3928dc-b1fd-4a1a-940e-82b8c55214e6
tenant_id: 3a55e7b5f5504649a2dfde7147383d02
extra_dhcp_opts: []
binding:vnic_type: normal
device_owner: compute:az_compute
mac_address: fa:16:3e:52:7a:f2
fixed_ips: [{u'subnet_id': u'94bf94c0-6568-4520-aee3-d12b5e472128', u'ip_address': u'10.0.0.2'}]
id: f47c62b0-dbd2-4faa-9cdd-8abc886ce08f
security_groups: [u'7c2b801b-4590-4a1f-9837-1cceb7f6d1d0']
device_id: c3522974-8a08-481c-87b5-fe3822f5c89c
## port_id = c4493802-4344-42bd-87a6-1b783f88609a
status: ACTIVE
name:
allowed_address_pairs: []
admin_state_up: True
network_id: ea3928dc-b1fd-4a1a-940e-82b8c55214e6
tenant_id: 3a55e7b5f5504649a2dfde7147383d02
extra_dhcp_opts: []
binding:vnic_type: normal
device_owner: compute:az_compute
mac_address: fa:16:3e:94:84:90
fixed_ips: [{u'subnet_id': u'94bf94c0-6568-4520-aee3-d12b5e472128', u'ip_address': u'10.0.0.4'}]
id: c4493802-4344-42bd-87a6-1b783f88609a
security_groups: [u'7c2b801b-4590-4a1f-9837-1cceb7f6d1d0']
device_id: 9365c842-9228-44a6-88ad-33d7389cda5f
```



#### sh
`EasyOVS> sh cmd`

执行系统命令。
```sh
EasyOVS> sh ls -l
total 48
drwxr-xr-x. 2 root root 4096 Apr  1 14:34 bin
drwxr-xr-x. 5 root root 4096 Apr  1 14:56 build
drwxr-xr-x. 2 root root 4096 Apr  1 14:56 dist
drwxr-xr-x. 2 root root 4096 Apr  1 14:09 doc
drwxr-xr-x. 4 root root 4096 Apr  1 14:56 easyovs
-rw-r--r--. 1 root root  660 Apr  1 14:56 easyovs.1
drwxr-xr-x. 2 root root 4096 Apr  1 14:56 easyovs.egg-info
-rw-r--r--. 1 root root 2214 Apr  1 14:53 INSTALL
-rw-r--r--. 1 root root 1194 Apr  1 14:53 Makefile
-rw-r--r--. 1 root root 3836 Apr  1 14:53 README.md
-rw-r--r--. 1 root root 1177 Apr  1 14:53 setup.py
drwxr-xr-x. 2 root root 4096 Apr  1 14:09 util
```

#### quit
输入 `^d` 或者 `quit` 命令来退出程序。

### 参数
#### -h
显示帮助信息。
```sh
$ easyovs -h
Usage: easyovs [options]
(type easyovs -h for details)

The easyovs utility creates operation CLI from the command line. It can run
given commands, invoke the EasyOVS CLI, and run tests.

Options:
  -h, --help            show this help message and exit
  -c, --clean           clean and exit
  -m CMD, --cmd=CMD     Run customized commands for tests.
  -v VERBOSITY, --verbosity=VERBOSITY
                        info|warning|critical|error|debug|output
  --version
```

#### -c
进行环境清理。

#### -m
不进入 CLI，直接执行给定的命令，显示结果。
例如
```sh
$ sudo easyovs -m "show br-int"
Intf                Port        Vlan    Type        vmIP            vmMAC
qvof47c62b0-db      2           1                   10.0.0.2        fa:16:3e:52:7a:f2
qvoc4493802-43      3           1                   10.0.0.4        fa:16:3e:94:84:90
br-int              LOCAL               internal
patch-tun           6                   patch

```

例如
```sh
$ sudo easyovs -m 'br-int dump'
ID TAB PKT       PRI   MATCH                                                       ACT
0  0   30        1     *                                                           NORMAL
1  23  0         0     *                                                           drop
```

#### -v
设置输出信息的日志级别，包括 debug，info，warn，error 等，方便进行调试。

#### --version
显示版本信息。
