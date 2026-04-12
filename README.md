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
auto ens192
iface ens192 inet static
address 172.16.1.1
netmask 255.255.255.240

auto ens224
iface ens224 inet static
address 172.16.2.1
netmask 255.255.255.240
</pre>

<pre>service networking restart</pre>

<pre>
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
</pre>

<pre>
apt install iptables iptables-persistent -y
</pre>

<p>nano /etc/iptables/iptables.sh</p>
<pre>
#!/bin/bash
iptables -F
iptables -X
iptables -t nat -F
iptables -t nat -X
iptables -P INPUT DROP
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p icmp -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A FORWARD -i ens192 -o ens224 -j ACCEPT
iptables -A FORWARD -i ens224 -o ens192 -j ACCEPT
iptables-save > /etc/iptables/rules.v4
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











