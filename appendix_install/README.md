# 附：安装配置
参考 https://github.com/yeasy/openstack-tool。

# DevStack 安装 OpenStack 多节点（Juno+Neutron+ML2+VXLAN）

目前安装 OpenStack 常见的方案有 Redhat 的 [RDO](https://openstack.redhat.com/) 和社区的 [DevStack](http://docs.openstack.org/developer/devstack/)。

当然，也可以手动安装，可以参考：[github.com/ChaimaGhribi/OpenStack-Juno-Installation/blob/master/OpenStack-Juno-Installation.rst](https://github.com/ChaimaGhribi/OpenStack-Juno-Installation/blob/master/OpenStack-Juno-Installation.rst)

其中，RDO 功能比较强大，运行也稳定，可以在一个节点上通过一个 answer 文件直接部署多个节点，搭建一套 OpenStack 环境。但是可惜，在 Ubuntu 上还不支持。

DevStack 支持 Ubuntu、Fedora 等环境，需要在每个节点上单独执行，适合进行实验。目前常见的教程一般都是讲解 DevStack 单节点安装。本文讲解最新的 Juno 版本在多节点上的安装过程。

# 网络环境
两台机器，分为控制节点（同时也作为网络节点）和计算节点。
## 控制节点
eth0: 9.186.100.77/24 作为管理网络（同时也是公共网络）。
eth1: 10.0.100.77/24 作为内部网络接口。

## 计算节点
eth0: 9.186.100.88/24 作为管理网络（同时也是公共网络）。
eth1: 10.0.100.88/24 作为内部网络接口。

# 配置 stack 用户

创建 stack 用户
```
sudo groupadd stack
sudo useradd -g stack -s /bin/bash -d /opt/stack -m stack
```

添加 stack 用户权限。

```
sudo echo "stack ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
```

切换到 stack 用户
```
sudo su - stack
```

# 下载代码

下载 devstack 代码，并切换到 stable/juno 分支。
```
sudo apt-get install git -y
git clone https://git.openstack.org/openstack-dev/devstack -b stable/juno
```

# 编写运行配置文件

在 devstack 根目录下，编写 local.conf。

控制节点的 local.conf

```sh
[[local|localrc]]

MULTI_HOST=true
HOST_IP=9.186.100.77 # management network

FIXED_RANGE=10.0.0.0/24 #tenant private network
NETWORK_GATEWAY=10.0.0.1
#FIXED_NETWORK_SIZE=4096

PUBLIC_INTERFACE=eth0  #public network
FLOATING_RANGE=9.186.100.128/25
PUBLIC_NETWORK_GATEWAY=9.186.100.1

TUNNEL_ENDPOINT_IP=10.0.100.77 # data network endpoint for vxlan

LOGFILE=/opt/stack/logs/stack.sh.log

# Credentials
ADMIN_PASSWORD=admin
MYSQL_PASSWORD=secret
RABBIT_PASSWORD=secret
SERVICE_PASSWORD=secret
SERVICE_TOKEN=abcdefghijklmnopqrstuvwxyz

# enable neutron-ml2-vxlan
disable_service n-net
enable_service q-svc,q-agt,q-dhcp,q-l3,q-meta,q-metering,q-lbaas,q-fwaas,q-vpn,neutron,tempest,heat

#OFFLINE=True

LOG_COLOR=False
LOGDIR=$DEST/logs
SCREEN_LOGDIR=$LOGDIR/screen

```

计算节点的 local.conf

```sh
[[local|localrc]]
MULTI_HOST=true

HOST_IP=9.186.100.88 # management network

FIXED_RANGE=10.0.0.0/24 # tenant private network
NETWORK_GATEWAY=10.0.0.1
#FIXED_NETWORK_SIZE=4096

TUNNEL_ENDPOINT_IP=10.0.100.88 # data network endpoint for vxlan

# Credentials
ADMIN_PASSWORD=admin
MYSQL_PASSWORD=secret
RABBIT_PASSWORD=secret
SERVICE_PASSWORD=secret
SERVICE_TOKEN=abcdefghijklmnopqrstuvwxyz

# Service information
SERVICE_HOST=9.186.100.77
MYSQL_HOST=$SERVICE_HOST
RABBIT_HOST=$SERVICE_HOST
GLANCE_HOSTPORT=$SERVICE_HOST:9292
Q_HOST=$SERVICE_HOST
KEYSTONE_AUTH_HOST=$SERVICE_HOST
KEYSTONE_SERVICE_HOST=$SERVICE_HOST

CEILOMETER_BACKEND=mongodb
DATABASE_TYPE=mysql

ENABLED_SERVICES=n-cpu,q-agt,neutron

# vnc config
NOVA_VNC_ENABLED=True
NOVNCPROXY_URL="http://$SERVICE_HOST:6080/vnc_auto.html"
VNCSERVER_LISTEN=$HOST_IP
VNCSERVER_PROXYCLIENT_ADDRESS=$VNCSERVER_LISTEN

#OFFLINE=True

LOG_COLOR=False
LOGDIR=$DEST/logs
SCREEN_LOGDIR=$LOGDIR/screen
#LOGFILE=/opt/stack/logs/stack.sh.log

```

# 执行配置

执行命令。
```
./stack.sh
```

会输出各项操作的结果。日志会写到 stack.sh.log 文件。成功后最后会打印出相关信息。

控制节点上

```
Horizon is now available at http://9.186.100.77/
Keystone is serving at http://9.186.100.77:5000/v2.0/
Examples on using novaclient command line is in exercise.sh
The default users are: admin and demo
The password: admin
This is your host ip: 9.186.100.77
```
执行成功后，在控制节点上，将物理网卡 eth0 绑定到 br-ex 网桥上。
```sh
sudo ifconfig eth0 0.0.0.0; sudo ovs-vsctl add-port br-ex eth0; sudo ifconfig br-ex 9.186.100.77/24; sudo route add default gw 9.186.100.1
```

# 其它事项

## 停止服务
停止 OpenStack 的各个服务（DevStack 默认每个服务都在一个 screen 中以进程方式运行，可以通过 screen -x stack 进入查看）。
```sh
./unstack.sh
```

## 清除安装。
```sh
./clean.sh
```

## 手动清除
有时候有些文件可能清除不干净，手动执行
```
sudo rm -rf /etc/libvirt/qemu/inst*
sudo virsh list | grep inst | awk '{print $1}' | xargs -n1 virsh destroy
```
## 依赖版本
安装过程中可能会报某些包版本不满足的问题，手动安装后。

例如报 six 包版本过低。

```
sudo pip install six --upgrade
```

## screen 操作
ctrl+a+" 显示 session 列表。
ctrl+a+d 挂起到后台。
