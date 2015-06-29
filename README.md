深入理解 Neutron -- OpenStack 网络实现
============
[Neutron](https://wiki.openstack.org/wiki/Neutron) 是 OpenStack 项目中负责提供网络服务的组件，它基于软件定义网络的思想，实现了网络虚拟化下的资源管理。

本书将剖析 Neutron 组件的原理和实现。

最新版本在线阅读：[GitBook](https://www.gitbook.io/book/yeasy/openstack_understand_Neutron)。

本书源码在 Github 上维护，欢迎参与： [https://github.com/yeasy/openstack_understand_Neutron](https://github.com/yeasy/openstack_understand_Neutron)。

感谢所有的 [贡献者](https://github.com/yeasy/openstack_understand_Neutron/graphs/contributors)。

## 更新历史:
* V0.9: 2015-06-29
	* 添加对 DVR 更多细节分析。
* V0.8: 2015-03-24
	* 添加 LBaaS 服务分析；
* V0.7: 2015-03-23
	* 添加 VXLAN 模式分析；
	* 添加 DVR 服务分析。
* V0.6: 2014-04-04
	* 修正系统结构的图表；
    * 修正部分描述。
* V0.5: 2014-03-24
	* 添加对 vlan 模式下具体规则的分析；
	* 更新 GRE 模式下 answerfile 的 IP 信息。
* V0.4: 2014-03-20
	* 添加对安全组实现的完整分析，添加整体逻辑图表；
	* 添加 vlan 模式下的 RDO answer 文件（更新 IP 信息）。
* V0.3: 2014-03-10
	* 添加 GRE 模式下对流表规则分析；
	* 添加 GRE 模式下的 answer 文件。
*V0.2: 2014-03-06
	* 修正图表引用错误；
	* 添加对 GRE 模式下流表细节分析。
* V0.1: 2014-02-20
	* 开始整体结构。


## 参加步骤
* 在 GitHub 上 `fork` 到自己的仓库，如 `user/openstack_understand_Neutron`，然后 `clone` 到本地，并设置用户信息。
```
$ git clone git@github.com:user/openstack_understand_Neutron.git
$ cd openstack_understand_Neutron
$ git config user.name "User"
$ git config user.email user@email.com
```

* 修改代码后提交，并推送到自己的仓库。
```
$ #do some change on the content
$ git commit -am "Fix issue #1: change helo to hello"
$ git push
```

* 在 GitHub 网站上提交 pull request。
* 定期使用项目仓库内容更新自己仓库内容。
```
$ git remote add upstream https://github.com/yeasy/openstack_understand_Neutron
$ git fetch upstream
$ git checkout master
$ git rebase upstream/master
$ git push -f origin master
```
