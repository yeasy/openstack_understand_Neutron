## FORWARD
FORWARD chain上主要实现安全组的功能。用户在配置缺省安全规则时候（例如允许ssh到vm，允许ping到vm），影响该chain。
```sh
#iptables --line-numbers -vnL FORWARD
Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
num   pkts bytes target     prot opt in     out     source               destination
1    16203 5342K neutron-filter-top  all  --  *      *       0.0.0.0/0            0.0.0.0/0
2    16203 5342K neutron-openvswi-FORWARD  all  --  *      *       0.0.0.0/0            0.0.0.0/0
3        0     0 REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0           reject-with icmp-host-prohibited
```
同样跳转到neutron-filter-top，无规则。跳转到neutron-openvswi-FORWARD。
```sh
#iptables --line-numbers -vnL neutron-openvswi-FORWARD
Chain neutron-openvswi-FORWARD (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1     8170 2630K neutron-openvswi-sg-chain  all  --  *      *       0.0.0.0/0            0.0.0.0/0           PHYSDEV match --physdev-out tap583c7038-d3 --physdev-is-bridged
2     8156 2729K neutron-openvswi-sg-chain  all  --  *      *       0.0.0.0/0            0.0.0.0/0           PHYSDEV match --physdev-in tap583c7038-d3 --physdev-is-bridged
```
neutron-openvswi-FORWARD将匹配所有进出tap-XXX端口的流量。
```sh
#iptables --line-numbers -vnL neutron-openvswi-sg-chain
Chain neutron-openvswi-sg-chain (2 references)
num   pkts bytes target     prot opt in     out     source               destination
1     8170 2630K neutron-openvswi-i583c7038-d  all  --  *      *       0.0.0.0/0            0.0.0.0/0           PHYSDEV match --physdev-out tap583c7038-d3 --physdev-is-bridged
2     8156 2729K neutron-openvswi-o583c7038-d  all  --  *      *       0.0.0.0/0            0.0.0.0/0           PHYSDEV match --physdev-in tap583c7038-d3 --physdev-is-bridged
3    12442 4163K ACCEPT     all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
如果是网桥从tap-XXX端口发出到VM的流量，则跳转到neutron-openvswi-i9LETTERID；如果是从tap-XXX端口进入到网桥的（即vm发出来的）流量，则跳转到neutron-openvswi-o9LETTERID。
```sh
#iptables --line-numbers -vnL neutron-openvswi-i583c7038-d
Chain neutron-openvswi-i583c7038-d (1 references)
num   pkts bytes target     prot opt in     out     source               destination
1        0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0           state INVALID
2      400 43350 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED
3        1    60 RETURN     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0           tcp dpt:22
4        1    84 RETURN     icmp --  *      *       0.0.0.0/0            0.0.0.0/0
5     3885 1391K RETURN     udp  --  *      *       192.168.0.3          0.0.0.0/0           udp spt:67 dpt:68
6     3885 1197K neutron-openvswi-sg-fallback  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
neutron-openvswi-i9LETTERID允许安全组中配置的策略（允许ssh、ping等）和dhcp reply通过。默认的neutron-openvswi-sg-fallback将drop所有流量。
```sh
#iptables --line-numbers -vnL neutron-openvswi-o583c7038-d
Chain neutron-openvswi-o583c7038-d (2 references)
num   pkts bytes target     prot opt in     out     source               destination
1     3886 1197K RETURN     udp  --  *      *       0.0.0.0/0            0.0.0.0/0           udp spt:68 dpt:67
2     4274 1533K neutron-openvswi-s583c7038-d  all  --  *      *       0.0.0.0/0            0.0.0.0/0
3        0     0 DROP       udp  --  *      *       0.0.0.0/0            0.0.0.0/0           udp spt:67 dpt:68
4        0     0 DROP       all  --  *      *       0.0.0.0/0            0.0.0.0/0           state INVALID
5     3963 1507K RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0           state RELATED,ESTABLISHED
6      311 25752 RETURN     all  --  *      *       0.0.0.0/0            0.0.0.0/0
7        0     0 neutron-openvswi-sg-fallback  all  --  *      *       0.0.0.0/0            0.0.0.0/0
```
neutron-openvswi-o9LETTERID将跳转到neutron-openvswi-s583c7038-d，允许DHCP Request和匹配VM的源IP和源MAC的流量通过。
