## 配置
要启用 DVR，比较简单，分别在各个节点的网络配置文件上做如下修改或添加。

### Neutron Server
`/etc/neutron/neutron.conf`

```sh
router_distributed = True
```

### L3 Agent
`/etc/neutron/l3_agent.ini`

```sh
agent_mode = [dvr_snat | dvr | legacy ]
```
网络节点上配置为 dvr_snat，计算节点上配置为 dvr。


### L2 Agent
`/etc/neutron/plugins/ml2/ml2_conf.ini`

```sh
[ml2]
mechanism_drivers = openvswitch,linuxbridge,l2population

[agent]
l2_population = True
tunnel_types = vxlan
enable_distributed_routing = True
```
