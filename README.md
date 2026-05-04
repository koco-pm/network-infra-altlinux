# Настройка сетевой инфраструктуры на примере Alt Linux 10.1

## ISP
Настройка интерфейсов через `nmtui`:
```bash
nmtui
```
- **hostname** - isp
- **ens18** – dhcp
- **ens19** – 172.16.1.1/28
- **ens20** – 172.16.2.1/28

```bash
exec bash
systemctl restart NetworkManager
```

Маршрутизация и NAT:
```bash
nano /etc/net/sysctl.conf
```
```bash
net.ipv4.ip_forward = 1
```
```bash
sysctl -p /etc/net/sysctl.conf

iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables-save >> /etc/sysconfig/iptables
systemctl enable --now iptables
```

## HQ-RTR
Настройка интерфейсов через `nmtui`:
```bash
nmtui
```
- **hostname** - hq-rtr.au-team.irpo
- **ens18** – 172.16.1.2/28
  + **gateway** - 172.16.1.1
  + **DNS** - 77.88.8.8
- **ens19** – x
  + **ens19.100** - 192.168.10.1/27
  + **ens19.200** - 192.168.20.1/28
  + **ens19.999** - 192.168.99.1/29

```bash
exec bash
systemctl restart NetworkManager
```

Маршрутизация и NAT:
```bash
nano /etc/net/sysctl.conf
```
```bash
net.ipv4.ip_forward = 1
```
```bash
sysctl -p /etc/net/sysctl.conf

iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables -A FORWARD -i ens19.100 -j ACCEPT
iptables -A FORWARD -i ens19.200 -j ACCEPT
iptables -A FORWARD -i ens19.999 -j ACCEPT
iptables-save >> /etc/sysconfig/iptables
systemctl enable --now iptables
```

GRE-туннель до BR-RTR:
```bash
nano /etc/rc.d/rc.local
```
```bash
#!/bin/bash
ip tunnel add tun1 mode gre remote 172.16.2.2 local 172.16.1.2 ttl 255 key 12345678
ip link set tun1 up
ip addr add 192.168.111.1/30 dev tun1
```
```bash
chmod +x /etc/rc.d/rc.local
reboot
```

Настройка OSPF:
```bash
systemctl enable --now frr
```
```bash
nano /etc/frr/daemons
```
```bash
zebra=yes
staticd=yes
ospfd=yes
```
>Пример файла конфигурации можно скачать с http-сервера:  
>wget http://88.201.141.149/hq-rtr/frr.conf

```bash
nano /etc/frr/frr.conf
```
```bash
ip forwarding
router ospf
 network 192.168.10.0/27 area 0
 network 192.168.20.0/28 area 0
 network 192.168.99.0/29 area 0
 network 192.168.111.0/30 area 0
interface tun1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 12345678
```
```bash
systemctl restart frr
```

DHCP-сервер для подсети 192.168.20.0/28:
```bash
systemctl enable --now dhcpd
```
>Пример файла конфигурации можно скачать с http-сервера:  
>wget http://88.201.141.149/dhcpd.conf

```bash
nano /etc/dhcp/dhcpd.conf
```
```bash
ddns-update-style none;
subnet 192.168.20.0 netmask 255.255.255.240 {
  option routers 192.168.20.1;
  option subnet-mask 255.255.255.240;
  option domain-name "au-team.irpo";
  option domain-name-servers 192.168.10.2;
  range 192.168.20.2 192.168.20.10;
  default-lease-time 21600;
  max-lease-time 43200;
  host HQ-CLI {
    hardware ethernet <MAC-адрес вида 11:22:33...>;
    fixed-address 192.168.20.2;
  }
}
```
```bash
nano /etc/sysconfig/dhcpd
```
```bash
DHCPDARGS=ens19.200
```
```bash
systemctl restart dhcpd
```

Пользователь net_admin:
```bash
useradd -m -s /bin/bash net_admin
passwd net_admin  #P@ssw0rd
usermod -aG wheel net_admin
```
```bash
nano /etc/sudoers
```
```bash
net_admin ALL=(ALL) NOPASSWD:ALL
```

## HQ-SRV
Настройка интерфейсов через `nmtui`:
```bash
nmtui
```
- **hostname** - hq-rtr.au-team.irpo
- **ens18** – 192.168.10.2/27
  + **gateway** - 192.168.10.1
  + **DNS** - 77.88.8.8
```bash
exec bash
systemctl restart NetworkManager
```

DNS-сервер:
```bash
systemctl enable --now bind
```
>Пример файла конфигурации можно скачать с http-сервера:  
>wget http://88.201.141.149/options.conf
```bash
rm -rf /etc/bind/options.conf
nano /etc/bind/options.conf
```
```bash
options {
  directory "/etc/bind/zone";
  recursion yes;
  allow-query { any; };
  forwarders {
    77.88.8.8;
  };
  dnssec-validation auto;
  listen-on { any; };
};
```
>Пример файла конфигурации можно скачать с http-сервера:  
>wget http://88.201.141.149/local.conf
```bash
nano /etc/bind/local.conf
```
```bash
zone "au-team.irpo" {
  type master;
  file "/etc/bind/zone/au-team.irpo";
};
zone "10.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/zone/db.192.168.10";
};
zone "20.168.192.in-addr.arpa" {
  type master;
  file "/etc/bind/zone/db.192.168.20";
};
```
>Пример файла конфигурации можно скачать с http-сервера:  
>wget http://88.201.141.149/au-team.irpo
```bash
nano /etc/bind/zone/au-team.irpo
```
```bash
$TTL 86400
@       IN SOA  ns.au-team.irpo. admin.au-team.irpo. (
        2025100801  ; Serial
        3600        ; Refresh
        1800        ; Retry
        604800      ; Expire
        86400 )     ; Minimum TTL
@       IN NS   ns.au-team.irpo.
ns      IN A    192.168.10.2
hq-rtr  IN A    192.168.10.1
br-rtr  IN A    172.16.2.2
hq-srv  IN A    192.168.10.2
hq-cli  IN A    192.168.20.2
br-srv  IN A    192.168.30.2
docker  IN A    172.16.1.1
web     IN A    172.16.2.1
```
>Пример файла конфигурации можно скачать с http-сервера:  
>wget http://88.201.141.149/db.192.168.20
```bash
nano /etc/bind/zone/db.192.168.20
```
```bash
$TTL 86400
@ IN SOA ns.au-team.irpo. admin.au-team.irpo. (
  2025100801 3600 1800 604800 86400 )
@ IN NS  ns.au-team.irpo.
2 IN PTR hq-cli.au-team.irpo.
```
>Пример файла конфигурации можно скачать с http-сервера:  
>wget http://88.201.141.149/db.192.168.10
```bash
nano /etc/bind/zone/db.192.168.10
```
```bash
$TTL 86400
@ IN SOA ns.au-team.irpo. admin.au-team.irpo. (
  2025100801 3600 1800 604800 86400 )
@ IN NS  ns.au-team.irpo.
1 IN PTR hq-rtr.au-team.irpo.
2 IN PTR hq-srv.au-team.irpo.
```

SSH-сервер:
```bash
systemctl enable --now sshd
```
```bash
nano /etc/openssh/sshd_config
```
```bash
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh_banner
```
```bash
nano /etc/ssh_banner
```
```bash
Authorized access only
```
```bash
systemctl restart sshd
```
```bash
useradd -u 2026 -m -s /bin/bash sshuser
passwd sshuser  # P@ssw0rd
usermod -aG wheel sshuser
```
```bash
nano /etc/sudoers
```
```bash
sshuser ALL=(ALL) NOPASSWD:ALL
```

## HQ-CLI
Настройка интерфейсов через `NetworkManager`:
- **ens18** – dhcp
```bash
su -
hostnamectl set-hostname hq-cli.au-team.irpo
exec bash
systemctl restart NetworkManager
```

## BR-RTR
Настройка интерфейсов через `nmtui`:
```bash
nmtui
```
- **hostname** - br-rtr.au-team.irpo
- **ens18** – 172.16.2.2/28
  + **gateway** - 172.16.2.1
  + **DNS** - 77.88.8.8
- **ens19** - 192.168.30.1/28
```bash
exec bash
systemctl restart NetworkManager
```

Маршрутизация и NAT:
```bash
nano /etc/net/sysctl.conf
```
```bash
net.ipv4.ip_forward = 1
```
```bash
sysctl -p /etc/net/sysctl.conf

iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables -A FORWARD -i ens19 -j ACCEPT
iptables-save >> /etc/sysconfig/iptables
systemctl enable --now iptables
```

GRE-туннель до HQ-RTR:
```bash
nano /etc/rc.d/rc.local
```
```bash
#!/bin/bash
ip tunnel add tun1 mode gre remote 172.16.1.2 local 172.16.2.2 ttl 255 key 12345678
ip link set tun1 up
ip addr add 192.168.111.2/30 dev tun1
```
```bash
chmod +x /etc/rc.d/rc.local
reboot
```

Настройка OSPF:
```bash
systemctl enable --now frr
```
>Пример файла конфигурации можно скачать с http-сервера:  
>wget http://88.201.141.149/br-rtr/frr.conf
```bash
nano /etc/frr/daemons
```
```bash
zebra=yes
staticd=yes
ospfd=yes
```
```bash
nano /etc/frr/frr.conf
```
```bash
router ospf
 network 192.168.30.0/28 area 0
 network 192.168.111.0/30 area 0
 area 0 authentication
interface tun1
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 12345678
ip forwarding
```
```bash
systemctl restart frr
```

Пользователь net_admin:
```bash
useradd -m -s /bin/bash net_admin
passwd net_admin  #P@ssw0rd
usermod -aG wheel net_admin
```
```bash
nano /etc/sudoers
```
```bash
net_admin ALL=(ALL) NOPASSWD:ALL
```

## BR-SRV
Настройка интерфейсов через `nmtui`:
```bash
nmtui
```
- **hostname** - br-srv.au-team.irpo
- **ens18** – 192.168.30.2/28
  + **gateway** - 192.168.10.1
  + **DNS** - 77.88.8.8
```bash
exec bash
systemctl restart NetworkManager
```

SSH-сервер:
```bash
systemctl enable --now sshd
```
```bash
nano /etc/openssh/sshd_config
```
```bash
Port 2026
AllowUsers sshuser
MaxAuthTries 2
Banner /etc/ssh_banner
```
```bash
nano /etc/ssh_banner
```
```bash
Authorized access only
```
```bash
systemctl restart sshd
```
```bash
useradd -u 2026 -m -s /bin/bash sshuser
passwd sshuser  # P@ssw0rd
usermod -aG wheel sshuser
```
```bash
nano /etc/sudoers
```
```bash
sshuser ALL=(ALL) NOPASSWD:ALL
```
