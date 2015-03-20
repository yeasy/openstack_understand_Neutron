## DHCP 服务
dhcp服务是通过dnsmasq进程（轻量级服务器，可以提供dns、dhcp、tftp等服务）来实现的，该进程绑定到dhcp名字空间中的br-int的接口上。可以查看相关的进程。
```sh
# ps -fe | grep 88b1609c-68e0-49ca-a658-f1edff54a264
nobody   23195     1  0 Oct26 ?        00:00:00 dnsmasq --no-hosts --no-resolv --strict-order --bind-interfaces --interface=ns-f14c598d-98 --except-interface=lo --pid-file=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/host --dhcp-optsfile=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/opts --dhcp-script=/usr/bin/neutron-dhcp-agent-dnsmasq-lease-update --leasefile-ro --dhcp-range=tag0,10.1.0.0,static,120s --conf-file= --domain=openstacklocal
root     23196 23195  0 Oct26 ?        00:00:00 dnsmasq --no-hosts --no-resolv --strict-order --bind-interfaces --interface=ns-f14c598d-98 --except-interface=lo --pid-file=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/pid --dhcp-hostsfile=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/host --dhcp-optsfile=/var/lib/neutron/dhcp/88b1609c-68e0-49ca-a658-f1edff54a264/opts --dhcp-script=/usr/bin/neutron-dhcp-agent-dnsmasq-lease-update --leasefile-ro --dhcp-range=tag0,10.1.0.0,static,120s --conf-file= --domain=openstacklocal
```
