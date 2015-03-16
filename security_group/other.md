## 其它
安全组在Havana版本中，默认是开启的，如果安装完毕后发现找不到qbr-*网桥，则可以检查在nova.conf里面是否设置以下内容：
```sh
libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
```
