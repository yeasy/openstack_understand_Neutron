## 实现细节

以 OpenvSwitch plugin 为例，主要在 `neutron\plugins\openvswitch\agent\ovs_dvr_neutron_agent.py` 文件中。

OVSDVRNeutronAgent 类是本地处理的 agent，启动后会在三个网桥上添加初始化的规则，主要过程如下。
```py
def setup_dvr_flows(self):
    self.setup_dvr_flows_on_integ_br()
    self.setup_dvr_flows_on_tun_br()
    self.setup_dvr_flows_on_phys_br()
    self.setup_dvr_mac_flows_on_all_brs()
```

其中，br-int 是本地交换网桥；br-tun 是跟其它计算节点通信的承载网桥；br-phy 是跟外部公共网络通信的网桥。


### integration 网桥规则添加
integrate 网桥负责本地同一子网内的网包交换。

主要过程为
```py
# Add a canary flow to int_br to track OVS restarts
        self.int_br.add_flow(table=constants.CANARY_TABLE, priority=0,
                             actions="drop")

        # Insert 'drop' action as the default for Table DVR_TO_SRC_MAC
        self.int_br.add_flow(table=constants.DVR_TO_SRC_MAC,
                             priority=1,
                             actions="drop")

        self.int_br.add_flow(table=constants.DVR_TO_SRC_MAC_VLAN,
                             priority=1,
                             actions="drop")

        # Insert 'normal' action as the default for Table LOCAL_SWITCHING
        self.int_br.add_flow(table=constants.LOCAL_SWITCHING,
                             priority=1,
                             actions="normal")

        for physical_network in self.bridge_mappings:
            self.int_br.add_flow(table=constants.LOCAL_SWITCHING,
                                 priority=2,
                                 in_port=self.int_ofports[physical_network],
                                 actions="drop")
```
首先，网桥上采用了多个流表，定义在 openvswitch/common 目录下的 constants.py。主要流表包括：
* LOCAL_SWITCHING 表：0，顾名思义，用于本地的交换。
* CANARY 表：23
* DVR_TO_SRC_MAC 表：1
* DVR_TO_SRC_MAC_VLAN 表：2

上面的代码配置 LOCAL_SWITCHING 表默认规则为 NORMAL（作为本地的正常学习交换机），并不允许宿主机本地端口进来的包。配置其它表默认行为为丢弃。

### tunnel 网桥规则添加
主要过程为
```py
        self.tun_br.add_flow(priority=1,
                             in_port=self.patch_int_ofport,
                             actions="resubmit(,%s)" %
                             constants.DVR_PROCESS)

        # table-miss should be sent to learning table
        self.tun_br.add_flow(table=constants.DVR_NOT_LEARN,
                             priority=0,
                             actions="resubmit(,%s)" %
                             constants.LEARN_FROM_TUN)

        self.tun_br.add_flow(table=constants.DVR_PROCESS,
                             priority=0,
                             actions="resubmit(,%s)" %
                             constants.PATCH_LV_TO_TUN)
```
主要流表包括：
* DVR_PROCESS 表：1，顾名思义，用于 DVR 处理。
* DVR_NOT_LEARN 表：9。
* LEARN_FROM_TUN 表：10，学习从 tunnel 收到的网包。
* PATCH_LV_TO_TUN 表：2。

所有从 br-int 过来的网包，发到 DVR_PROCESS 表处理。

DVR_NOT_LEARN 表，默认将网包发给表 LEARN_FROM_TUN。

DVR_PROCESS 表，默认将网包发给表 PATCH_LV_TO_TUN。

### physical 网桥规则添加
主要过程为
```py
for physical_network in self.bridge_mappings:
            self.phys_brs[physical_network].add_flow(priority=2,
                in_port=self.phys_ofports[physical_network],
                actions="resubmit(,%s)" %
                constants.DVR_PROCESS_VLAN)
            self.phys_brs[physical_network].add_flow(priority=1,
                actions="resubmit(,%s)" %
                constants.DVR_NOT_LEARN_VLAN)
            self.phys_brs[physical_network].add_flow(
                table=constants.DVR_PROCESS_VLAN,
                priority=0,
                actions="resubmit(,%s)" %
                constants.LOCAL_VLAN_TRANSLATION)
            self.phys_brs[physical_network].add_flow(
                table=constants.LOCAL_VLAN_TRANSLATION,
                priority=2,
                in_port=self.phys_ofports[physical_network],
                actions="drop")
            self.phys_brs[physical_network].add_flow(
                table=constants.DVR_NOT_LEARN_VLAN,
                priority=1,
                actions="NORMAL")
```
对所有的连接到外部的物理网桥都添加类似规则。

主要流表包括：
* DVR_PROCESS_VLAN：1
* LOCAL_VLAN_TRANSLATION：2
* DVR_NOT_LEARN_VLAN：3。

默认从 br-int 来的所有网包发送给表 DVR_PROCESS_VLAN。

DVR_PROCESS_VLAN 表，默认发给表 LOCAL_VLAN_TRANSLATION。

LOCAL_VLAN_TRANSLATION 表丢弃从 br-int 来的所有网包。

其他所有网包发送给表 DVR_NOT_LEARN_VLAN。

表 DVR_NOT_LEARN_VLAN 默认规则为 NORMAL。

### 所有网桥上添加 mac 规则
主要过程为
```py
for physical_network in self.bridge_mappings:
    self.int_br.add_flow(table=constants.LOCAL_SWITCHING,
        priority=4,
        in_port=self.int_ofports[physical_network],
        dl_src=mac['mac_address'],
        actions="resubmit(,%s)" %
        constants.DVR_TO_SRC_MAC_VLAN)
    self.phys_brs[physical_network].add_flow(
        table=constants.DVR_NOT_LEARN_VLAN,
        priority=2,
        dl_src=mac['mac_address'],
        actions="output:%s" %
        self.phys_ofports[physical_network])

if self.enable_tunneling:
    # Table 0 (default) will now sort DVR traffic from other
    # traffic depending on in_port
    self.int_br.add_flow(table=constants.LOCAL_SWITCHING,
                         priority=2,
                         in_port=self.patch_tun_ofport,
                         dl_src=mac['mac_address'],
                         actions="resubmit(,%s)" %
                         constants.DVR_TO_SRC_MAC)
    # Table DVR_NOT_LEARN ensures unique dvr macs in the cloud
    # are not learnt, as they may
    # result in flow explosions
    self.tun_br.add_flow(table=constants.DVR_NOT_LEARN,
                     priority=1,
                     dl_src=mac['mac_address'],
                     actions="output:%s" %
                     self.patch_int_ofport)
self.registered_dvr_macs.add(mac['mac_address'])
```
这里主要是添加对其他主机上的 DVR 网关发过来的网包的处理。
#### 外部网络为 vlan 的情况下
br-int 网桥上，表 LOCAL_SWITCHING 添加优先级为 4 的流，如果从外面过来的网包，MAC 源地址是其他的 DVR 网关地址，则扔给表 DVR_TO_SRC_MAC_VLAN 处理。

br-eth 网桥上，表 DVR_NOT_LEARN_VLAN 则添加优先级为 2 的规则，MAC 源地址是其他的 DVR 网关地址，则发给 br-int。

#### 外部网络为 tunnel 的情况下
类似，分别在 br-int 网桥的 LOCAL_SWITCHING 表和 br-tun 网桥的 DVR_NOT_LEARN 表添加流，优先级分别改为 2 和 1。
