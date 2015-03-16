# 概述
Neutron 的设计目标是实现“网络即服务”，为了达到这一目标，在设计上遵循了基于“软件定义网络”实现网络虚拟化的原则，在实现上充分利用了 Linux 系统上的各种网络相关的技术。

理解了 Linux 系统上的这些概念将有利于快速理解 Neutron 的原理和实现。

## 涉及的 Linux 网络技术
* bridge：网桥，Linux中用于表示一个能连接不同网络设备的虚拟设备，linux中传统实现的网桥类似一个hub设备，而ovs管理的网桥一般类似交换机。
* br-int：bridge-integration，综合网桥，常用于表示实现主要内部网络功能的网桥。
* br-ex：bridge-external，外部网桥，通常表示负责跟外部网络通信的网桥。
* GRE：General Routing Encapsulation，一种通过封装来实现隧道的方式。在openstack中一般是基于L3的gre，即original pkt/GRE/IP/Ethernet
* VETH：虚拟ethernet接口，通常以pair的方式出现，一端发出的网包，会被另一端接收，可以形成两个网桥之间的通道。
* qvb：neutron veth, Linux Bridge-side
* qvo：neutron veth, OVS-side
* TAP设备：模拟一个二层的网络设备，可以接受和发送二层网包。
* TUN设备：模拟一个三层的网络设备，可以接受和发送三层网包。
* iptables：Linux 上常见的实现安全策略的防火墙软件。
* Vlan：虚拟 Lan，同一个物理 Lan 下用标签实现隔离，可用标号为1-4094。
* VXLAN：一套利用 UDP 协议作为底层传输协议的 Overlay 实现。一般认为作为 VLan 技术的延伸或替代者。
* namespace：用来实现隔离的一套机制，不同 namespace 中的资源之间彼此不可见。
