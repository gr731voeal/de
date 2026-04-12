## Таблица IP-адресации

| Устройство | Интерфейс / Назначение | IP-адрес / Маска | Шлюз |
|------------|------------------------|------------------|------|
| **HQ-RTR** | к hq-srv | 192.168.100.1/26 | 172.16.1.1 |
| | к hq-cli | 192.168.200.1/28 | |
| | к isp | 172.16.1.2/28 | |
| | gre к br-rtr | 10.10.10.1/30 | |
| **BR-RTR** | к br-srv | 192.168.0.1/27 | 172.16.2.1 |
| | к isp | 172.16.2.2/28 | |
| | gre к hq-rtr | 10.10.10.2/30 | |
| **HQ-SRV** | к hq-rtr | 192.168.100.2/26 | 192.168.100.1 |
| **BR-SRV** | к br-rtr | 192.168.0.2/27 | 192.168.0.1 |
| **HQ-CLI** | к hq-rtr | 192.168.200.2/28 | 192.168.200.1 |
| **ISP** | к hq-rtr | 172.16.1.1/28 | |
| | к br-rtr | 172.16.2.1/28 | |

## ISP

<p>nano /etc/apt/sources.list</p>
<pre>Отключить диск</pre>

<p>nano etc/resolv.conf</p>
<pre>
nameserver 8.8.8.8
nameserver 192.168.100.2
</pre>

<pre>
hostnamectl set-hostname isp.au-team.irpo
newgrp
timedatectl set-timezone Asia/Tomsk
</pre>

<p>nano /etc/network/interfaces</p>
<pre>
auto ens224
iface ens224 inet static
address 172.16.1.1
netmask 255.255.255.240
&#10;
auto ens256
iface ens256 inet static
address 172.16.2.1
netmask 255.255.255.240
</pre>

<pre>service networking restart</pre>

<pre>
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
</pre>

<pre>apt install iptables iptables-persistent -y</pre>

<p>nano /etc/iptables/iptables.sh</p>
<pre>
#!/bin/bash
&#10;
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
&#10;
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
&#10;
iptables -t nat -A POSTROUTING -s 172.16.1.0/28 -o ens192 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 172.16.2.0/28 -o ens192 -j MASQUERADE
&#10;
iptables -A FORWARD -s 172.16.1.0/28 -i any -o ens192 -j ACCEPT
iptables -A FORWARD -s 172.16.2.0/28 -i any -o ens192 -j ACCEPT
iptables -A FORWARD -d 172.16.1.0/28 -i ens192 -o any -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -d 172.16.2.0/28 -i ens192 -o any -m state --state ESTABLISHED,RELATED -j ACCEPT
</pre>

<pre>
chmod +x /etc/iptables/iptables.sh
/etc/iptables/iptables.sh
systemctl restart iptables
service networking restart
</pre>

## HQ-RTR

<p>nano /etc/apt/sources.list</p>
<pre>Отключить диск</pre>

<p>nano etc/resolv.conf</p>
<pre>
nameserver 8.8.8.8
nameserver 192.168.100.2
</pre>

<pre>
hostnamectl set-hostname hq-rtr.au-team.irpo
newgrp
timedatectl set-timezone Asia/Tomsk
</pre>

<p>nano /etc/network/interfaces</p>
<pre>
auto ens192
iface ens192 inet static
address 172.16.1.2
netmask 255.255.255.240
gateway 172.16.1.1
</pre>

<pre>service networking restart</pre>

<pre>apt install vlan -y
modprobe 8021q
echo 8021q >> /etc/modules
echo ip_gre >> /etc/modules
</pre>

<p>nano /etc/network/interfaces</p>
<pre>
auto ens192
iface ens192 inet static
address 172.16.1.2
netmask 255.255.255.240
gateway 172.16.1.1
&#10;
auto ens224 
iface ens224 inet static 
address 192.168.100.1 
netmask 255.255.255.192
&#10;
auto ens224:1 
iface ens224:1 inet static 
address 192.168.200.1 
netmask 255.255.255.240
&#10;
auto ens224.100 
iface ens224.100 inet static 
address 192.168.100.3 
netmask 255.255.255.192 
vlan-raw-device ens224
&#10;
auto ens224.200 
iface ens224.200 inet static 
address 192.168.200.3 
netmask 255.255.255.240 
vlan-raw-device ens224:1
&#10;
auto tun0 
iface tun0 inet tunnel
address 10.10.0.1 
netmask 10.10.0.1
mode gre 
local 172.16.1.2
endpoint 172.16.2.2 
ttl 64
</pre>

<pre>service networking restart</pre>

<pre>apt install iptables iptables-persistent -y</pre>

<p>nano /etc/iptables/iptables.sh</p>
<pre>
#!/bin/bash
&#10;
iptables -F 
iptables -X
iptables -t nat -F 
iptables -t nat -X
&#10;
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT 
iptables -P OUTPUT ACCEPT
&#10;
iptables -t nat -A POSTROUTING -o ens192 -j MASQUERADE
&#10;
iptables -A FORWARD -i ens224.100 -o ens192 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT 
iptables -A FORWARD -i ens224.200 -o ens192 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT 
iptables -A FORWARD -i ens192 -o ens224.100 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT 
iptables -A FORWARD -i ens192 -o ens224.200 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
&#10;
iptables-save > /etc/iptables/rules.v4
</pre>

<pre>
chmod +x /etc/iptables/iptables.sh
/etc/iptables/iptables.sh
systemctl restart iptables
service networking restart
</pre>

<pre>
useradd -m -G sudo net_admin
passwd net_admin
</pre>

<pre>apt install rfr</pre>

<p>nano /etc/frr/daemons</p>
<pre>
ospfd=yes
</pre>

<pre>service frr restart</pre>

<p>nano /etc/frr/frr.conf</p>
<pre>
configure terminal
&#10;
interface tun0
ip ospf authentication
ip ospf authentication-key P@ssw0rd
no ip ospf network broadcast
no ip ospf passive
exit
router ospf
ospf router-id 1.1.1.1
passive-interface default
network 10.10.10.0/30 area 0
network 192.168.100.0/27 area 0
network 182.168.200.0/28 area 0
area 0 authentication
exit
end
write memory
</pre>










