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

<p>/etc/apt/sources.list</p>
<pre>Отключить диск</pre>

<p>etc/resolv.conf</p>
<pre>
nameserver 8.8.8.8
nameserver 192.168.100.2
</pre>

<pre>
hostnamectl set-hostname isp.au-team.irpo
newgrp
timedatectl set-timezone Asia/Tomsk
</pre>

<p>/etc/network/interfaces</p>
<pre>
auto ens192
iface ens192 inet static
    address 172.16.1.1
    netmask 255.255.255.240
</pre>
<pre>
auto ens224
iface ens224 inet static
    address 172.16.2.1
    netmask 255.255.255.240
</pre>

<pre>service networking restart</pre>









