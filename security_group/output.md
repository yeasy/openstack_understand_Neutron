## OUTPUT
```sh
#iptables --line-numbers -vnL OUTPUT
Chain OUTPUT (policy ACCEPT 965K packets, 149M bytes)
num   pkts bytes target     prot opt in     out     source               destination
1     481K  107M neutron-filter-top  all  --  *      *       0.0.0.0/0            0.0.0.0/0
2     481K  107M neutron-openvswi-OUTPUT  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
分别跳转到neutron-filter-top和neutron-openvswi-OUTPUT。
```sh
#iptables --line-numbers -vnL neutron-filter-top
Chain neutron-filter-top (2 references)
num   pkts bytes target     prot opt in     out     source               destination
1     497K  112M neutron-openvswi-local  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
跳转到neutron-openvswi-local。

```sh
#iptables --line-numbers -vnL neutron-openvswi-OUTPUT
Chain neutron-openvswi-OUTPUT (1 references)
num   pkts bytes target     prot opt in     out     source               destination
```
该chain目前无规则。

```sh
#iptables --line-numbers -vnL neutron-openvswi-local
Chain neutron-openvswi-local (1 references)
num   pkts bytes target     prot opt in     out     source               destination
```
该chain目前也无规则。
