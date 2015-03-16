## INPUT
```sh
#iptables --line-numbers -vnL INPUT
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1     360K   56M neutron-openvswi-INPUT  all  --  *      *       0.0.0.0/0            0.0.0.0/0
2    10583 2146K ACCEPT     tcp  --  *      *       192.168.122.100      0.0.0.0/0           multiport dports 5666 /* 001 nagios-nrpe incoming 192.168.122.100 */
3      846 50966 ACCEPT     tcp  --  *      *       192.168.122.100      0.0.0.0/0           multiport dports 5900:5999 /* 001 nova compute incoming 192.168.122.100 */
4    1033K  894M ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED
5      760 63840 ACCEPT     icmp --  *      *       0.0.0.0/0            0.0.0.0/0
6        1    60 ACCEPT     all  --  lo     *       0.0.0.0/0            0.0.0.0/0
7      977 58620 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0           state NEW tcp dpt:22
8     3899 1194K REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited
```
可以看到，跟安全组相关的规则被重定向到neutron-openvswi-INPUT。
查看其规则，只有一条。
```sh
#iptables --line-numbers -vnL neutron-openvswi-INPUT
Chain neutron-openvswi-INPUT (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 neutron-openvswi-o583c7038-d  all  --  *      *       0.0.0.0/0            0.0.0.0/0           PHYSDEV match --physdev-in tap583c7038-d3 --physdev-is-bridged
```
重定向到neutron-openvswi-o583c7038-d。
```sh
#iptables --line-numbers -vnL neutron-openvswi-o583c7038-d
Chain neutron-openvswi-o583c7038-d (2 references)
num   pkts bytes target     prot opt in     out     source               destination
1     3894 1199K RETURN     udp  --  *      *       0.0.0.0/0            0.0.0.0/0           udp spt:68 dpt:67
2     4282 1536K neutron-openvswi-s583c7038-d  all  --  *      *       0.0.0.0/0            0.0.0.0/0
3        0     0 DROP       udp  --  *      *       0.0.0.0/0            0.0.0.0/0           udp spt:67 dpt:68
4        0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0           state INVALID
5     3971 1510K RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED
6      311 25752 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
7        0     0 neutron-openvswi-sg-fallback  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
如果是vm发出的dhcp请求，直接通过，否则转到neutron-openvswi-s583c7038-d。
```sh
#iptables --line-numbers -vnL neutron-openvswi-s583c7038-d
Chain neutron-openvswi-s583c7038-d (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1     4284 1537K RETURN     all  --  *      *       192.168.0.2          0.0.0.0/0           MAC FA:16:3E:9C:DC:3A
2        0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
这条chain主要检查从vm发出来的网包，是否是openstack所分配的IP和MAC，如果不匹配，则禁止通过。这将防止利用vm上进行一些伪装地址的攻击。
