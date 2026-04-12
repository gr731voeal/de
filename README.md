- [Таблица IP-адресации](#таблица-ip-адресации)
- [ISP](#isp)
- [HQ-RTR](#hq-rtr)
- [BR-RTR](#br-rtr)
- [HQ-SRV](#hq-srv)
- [BR-SRV](#br-srv)
- [HQ-CLI](#hq-cli)
- [Картинка](#по-просьбе-сильно-желающих)

## Таблица IP-адресации

| Устройство | Интерфейс / Назначение | IP-адрес / Маска | Шлюз |
|------------|------------------------|------------------|------|
| **HQ-RTR** | к hq-srv | 192.168.100.1/27 | 172.16.1.1 |
| | к hq-cli | 192.168.200.1/28 | |
| | к isp | 172.16.1.2/28 | |
| | gre к br-rtr | 10.10.0.1/30 | |
| **BR-RTR** | к br-srv | 192.168.0.1/28 | 172.16.2.1 |
| | к isp | 172.16.2.2/28 | |
| | gre к hq-rtr | 10.10.0.2/30 | |
| **HQ-SRV** | к hq-rtr | 192.168.100.2/27 | 192.168.100.1 |
| **BR-SRV** | к br-rtr | 192.168.0.2/28 | 192.168.0.1 |
| **HQ-CLI** | к hq-rtr | 192.168.200.2/28 | 192.168.200.1 |
| **ISP** | к hq-rtr | 172.16.1.1/28 | |
| | к br-rtr | 172.16.2.1/28 | |

## ISP

<p>nano /etc/apt/sources.list</p>
<pre>Отключить диск</pre>

<p>nano etc/resolv.conf</p>
<pre>
nameserver 192.168.100.2
nameserver 8.8.8.8
</pre>

<pre>
hostnamectl set-hostname isp.au-team.irpo
newgrp
timedatectl set-timezone Asia/Tomsk
</pre>

<pre>
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
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
nameserver 192.168.100.2
nameserver 8.8.8.8
</pre>

<pre>
hostnamectl set-hostname hq-rtr.au-team.irpo
newgrp
timedatectl set-timezone Asia/Tomsk
</pre>

<pre>
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
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
echo 8021q >> /etc/modules
modprobe 8021q
echo ip_gre >> /etc/modules
modprobe ip_gre
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
    netmask 255.255.255.224
&#10;
auto ens224:1
iface ens224:1 inet static
    address 192.168.200.1
    netmask 255.255.255.240
&#10;
auto ens224.100
iface ens224 inet static 
    address 192.168.100.3
    netmask 255.255.255.224
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
    netmask 255.255.255.252
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
iptables -A INPUT -p gre -s 172.16.2.2 -d 172.16.1.2 -j ACCEPT
iptables -A OUTPUT -p gre -s 172.16.1.2 -d 172.16.2.2 -j ACCEPT
iptables -A FORWARD -i tun0 -j ACCEPT
iptables -A FORWARD -o tun0 -j ACCEPT
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

<pre>apt install frr</pre>

<p>nano /etc/frr/daemons</p>
<pre>
ospfd=yes
</pre>

<pre>service frr restart</pre>

<pre>vtysh</pre>
<pre>
configure terminal
&#10;
interface tun0
    ip ospf authentication
    ip ospf authentication-key P@ssw0rd
    no ip ospf network broadcast
    no ip ospf passive
    exit
&#10;
router ospf
    ospf router-id 1.1.1.1
    passive-interface default
    no passive-interface tun0
    network 10.10.0.0/30 area 0
    network 192.168.100.0/27 area 0
    network 192.168.200.0/28 area 0
    area 0 authentication
    exit
&#10;
end
write memory
</pre>

<pre>
systemctl restart frr
systemctl enable frr
</pre>

<pre>
apt install isc-dhcp-server -y
</pre>

<p>nano /etc/default/isc-dhcp-server</p>
<pre>
INTERFACESv4="ens224:1"
</pre>

<p>nano /etc/dhcp/dhcpd.conf</p>
<pre>
subnet 192.168.200.0 netmask 255.255.255.240 {
    range 192.168.200.4 192.168.200.14;
    option domain-name-servers 192.168.100.2;
    option domain-name "au-team.irpo";
    option routers 192.168.200.1;
    default-lease-time 600;
    max-lease-time 7200;
}
</pre>

<pre>
systemctl restart isc-dhcp-server
systemctl enable isc-dhcp-server
</pre>

## BR-RTR

<p>nano /etc/apt/sources.list</p>
<pre>Отключить диск</pre>

<p>nano etc/resolv.conf</p>
<pre>
nameserver 192.168.100.2
nameserver 8.8.8.8
</pre>

<pre>
hostnamectl set-hostname br-rtr.au-team.irpo
newgrp
timedatectl set-timezone Asia/Tomsk
</pre>

<pre>
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
</pre>

<pre>
echo ip_gre >> /etc/modules
modprobe ip_gre
</pre>

<p>nano /etc/network/interfaces</p>
<pre>
auto ens192
iface ens192 inet static
    address 172.16.2.2
    netmask 255.255.255.240
    gateway 172.16.2.1
&#10;
auto ens224
iface ens224 inet static
    address 192.168.0.1
    netmask 255.255.255.240
&#10;
auto tun0
iface tun0 inet tunnel
    address 10.10.0.2
    netmask 255.255.255.252
    mode gre
    local 172.16.2.2
    endpoint 172.16.1.2
    ttl 64
</pre>

<pre>service networking restart</pre>

<pre>
apt install iptables iptables-persistent -y
</pre>

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
iptables -A FORWARD -i ens224 -o ens192 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i ens192 -o ens224 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
&#10;
iptables -A INPUT -p gre -s 172.16.1.2 -d 172.16.2.2 -j ACCEPT
iptables -A OUTPUT -p gre -s 172.16.2.2 -d 172.16.1.2 -j ACCEPT
iptables -A FORWARD -i tun0 -j ACCEPT
iptables -A FORWARD -o tun0 -j ACCEPT
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

<pre>apt install frr</pre>

<p>nano /etc/frr/daemons</p>
<pre>
ospfd=yes
</pre>

<pre>service frr restart</pre>

<pre>vtysh</pre>
<pre>
configure terminal
&#10;
interface tun0
    ip ospf authentication
    ip ospf authentication-key P@ssw0rd
    no ip ospf network broadcast
    no ip ospf passive
    exit
&#10;
router ospf
    ospf router-id 2.2.2.2
    passive-interface default
    no passive-interface tun0
    network 10.10.0.0/30 area 0
    network 192.168.0.0/28 area 0
    area 0 authentication
    exit
&#10;
end
write memory
</pre>

<pre>
systemctl restart frr
systemctl enable frr
</pre>

<p>Проверка</p>
<pre>
vtysh -c "show ip ospf neighbor"
vtysh -c "show ip ospf route"
</pre>

## HQ-SRV

<p>nano /etc/apt/sources.list</p>
<pre>Отключить диск</pre>

<p>nano etc/resolv.conf</p>
<pre>
nameserver 127.0.0.1
nameserver 8.8.8.8
</pre>

<pre>
hostnamectl set-hostname hq-srv.au-team.irpo
newgrp
timedatectl set-timezone Asia/Tomsk
</pre>

<p>nano /etc/network/interfaces</p>
<pre>
auto ens192
iface ens192 inet static
    address 192.168.100.2
    netmask 255.255.255.224
    gateway 192.168.100.1
</pre>

<pre>service networking restart</pre>

<p>nano /etc/hosts</p>
<pre>
127.0.1.1        hq-srv.au-team.irpo
192.168.100.2    hq-srv.au-team.irpo
</pre>

<pre>
apt update
apt install bind9 bind9utils bind9-doc dnsutils -y
</pre>

<pre>
cp /etc/bind/db.local /etc/bind/db.au-team.irpo
cp /etc/bind/db.127 /etc/bind/db.192.168.100
cp /etc/bind/db.127 /etc/bind/db.192.168.0
cp /etc/bind/db.127 /etc/bind/db.192.168.200
</pre>
    
<p>nano /etc/bind/named.conf.options</p>
<pre>
forwarders {
        8.8.8.8;
        8.8.4.4;
        77.88.8.7;
        77.88.8.3;
    };
</pre>

<p>nano /etc/bind/named.conf.local</p>
<pre>
zone "au-team.irpo" {
    type master;
    file "/etc/bind/db.au-team.irpo";
};

zone "100.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.100";
};

zone "0.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.0";
};

zone "200.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.200";
};
</pre>

<p>nano /etc/bind/db.au-team.irpo</p>
<pre>
$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. admin.au-team.irpo. (
                  2026041201  ; Serial
                        604800  ; Refresh
                         86400  ; Retry
                       2419200  ; Expire
                        604800 ) ; Negative Cache TTL

@       IN      NS      hq-srv.au-team.irpo.

; A Records
hq-rtr  IN      A       192.168.100.1
br-rtr  IN      A       192.168.0.1
hq-srv  IN      A       192.168.100.2
hq-cli  IN      A       192.168.200.2
br-srv  IN      A       192.168.0.2
docker  IN      A       172.16.1.1
web     IN      A       172.16.2.1
</pre>

<p>nano /etc/bind/db.192.168.100</p>
<pre>
$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. admin.au-team.irpo. (
                  2026041201  ; Serial
                        604800  ; Refresh
                         86400  ; Retry
                       2419200  ; Expire
                        604800 ) ; Negative Cache TTL

@       IN      NS      hq-srv.au-team.irpo.

1       IN      PTR     hq-rtr.au-team.irpo.
2       IN      PTR     hq-srv.au-team.irpo.
</pre>

<p>nano /etc/bind/db.192.168.0</p>
<pre>
$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. admin.au-team.irpo. (
                  2026041201  ; Serial
                        604800  ; Refresh
                         86400  ; Retry
                       2419200  ; Expire
                        604800 ) ; Negative Cache TTL

@       IN      NS      hq-srv.au-team.irpo.

1       IN      PTR     br-rtr.au-team.irpo.
2       IN      PTR     br-srv.au-team.irpo.
</pre>

<p>nano /etc/bind/db.192.168.200</p>
<pre>
$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. admin.au-team.irpo. (
                  2026041201  ; Serial
                        604800  ; Refresh
                         86400  ; Retry
                       2419200  ; Expire
                        604800 ) ; Negative Cache TTL

@       IN      NS      hq-srv.au-team.irpo.

2       IN      PTR     hq-cli.au-team.irpo.
</pre>

<pre>
systemctl restart bind9
systemctl enable bind9
systemctl status bind9
</pre>

<p>Проверка</p>
<pre>
host hq-rtr.au-team.irpo
host br-srv.au-team.irpo
host 192.168.100.1
host 192.168.0.2
</pre>

<pre>
useradd -m -u 2026 -G sudo sshuser
passwd sshuser
</pre>

<p>nano /etc/ssh/sshd_config</p>
<pre>
Port 2026
PermitRootLogin no
PasswordAuthentication yes
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh/bannermotd
</pre>

<p>nano /etc/ssh/bannermotd</p>
<pre>Authorized access only</pre>

<pre>service sshd restart</pre>

## BR-SRV

<p>nano /etc/apt/sources.list</p>
<pre>Отключить диск</pre>

<p>nano etc/resolv.conf</p>
<pre>
nameserver 192.168.100.2
nameserver 8.8.8.8
</pre>

<pre>
hostnamectl set-hostname br-srv.au-team.irpo
newgrp
timedatectl set-timezone Asia/Tomsk
</pre>

<p>nano /etc/network/interfaces</p>
<pre>
auto ens192
iface ens192 inet static
    address 192.168.0.2
    netmask 255.255.255.224
    gateway 192.168.0.1
</pre>

<pre>service networking restart</pre>

<pre>
useradd -m -u 2026 -G sudo sshuser
passwd sshuser
</pre>

<p>nano /etc/ssh/sshd_config</p>
<pre>
Port 2026
PermitRootLogin no
PasswordAuthentication yes
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh/bannermotd
</pre>

<p>nano /etc/ssh/bannermotd</p>
<pre>Authorized access only</pre>

<pre>service sshd restart</pre>

## HQ-CLI

<pre>
hostnamectl set-hostname hq-cli.au-team.irpo
newgrp
timedatectl set-timezone Asia/Tomsk
</pre>

<pre>service networking restart</pre>

<p>nano /etc/resolv.conf (если DHCP не передает DNS)</p>
<pre>nameserver 192.168.100.2</pre>


## По просьбе сильно желающих

<p align="center">
    <img width="553" height="386" alt="image" src="https://github.com/user-attachments/assets/d7246a28-74ee-45fa-a2ba-d67af781eb9d" />
</p>
