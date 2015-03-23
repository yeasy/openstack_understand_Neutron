## 配置
要启用 DVR，比较简单，分别在各个节点的网络配置文件上做如下修改。

### Neutron Server:
```sh
router_distributed = True
```

### L3 Agent:
```sh
agent_mode = [dvr_snat | dvr | legacy ]
```
网络节点上配置为 dvr_snat，计算节点上配置为 dvr。


### L2 Agent:
```sh
enable_distributed_routing = True
```
