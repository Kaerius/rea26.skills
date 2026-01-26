Сеть построена на трёх маршрутизаторах **EcoRouter OS** с использованием **MPLS/LDP + BGP + VPLS**:

---

### 🌐 **Внутренняя связность (офисы)**  
- **MPLS-GW-CORE** — P-устройство, **MPLS-GW-BR / MPLS-GW-CR** — PE.  
- Между ними работает **OSPF + LDP** для транспорта меток.  
- **VPLS `office-lan` (ID 100)** обеспечивает **прозрачный L2-сегмент** между офисами: клиенты в BR и CR находятся в одном широковещательном домене.

---

### 🌍 **Доступ в Интернет**  
- На **MPLS-GW-CORE** создан **VPLS `INET` (ID 10)**, подключённый к внешнему интерфейсу (`ge2`).  
- Этот VPLS реплицируется на **BR и CR**, где завершается как **VLAN 5** на серверах (например, `CR-SRV`).  
- Сервер получает IP из **192.168.122.0/24** по DHCP от провайдера.  
- На сервере включён **IP forwarding + NAT (masquerade)**, что делает его **шлюзом в Интернет** для локальной сети (192.168.1.0/24).  
- Маршрут по умолчанию распространяется через **BGP `default-originate`** с CORE → PE, обеспечивая полную маршрутизацию.

---

### 🔒 **Управление и безопасность**  
- SSH доступ разрешён только из **VLAN 100 (192.168.100.0/24)** к офисным роутерам; к CORE — от всех CLI.  
- Настроены **IP SLA** (проверка ISP-SRV), **SNMPv3**, **резервное копирование на TFTP**.  
- Все клиенты и серверы могут выходить в Интернет через NAT-сервер.

---


# Настройка EcoRouter

[Сохраненный конфиг с роутеров](https://pastebin.com/h52ivJCx)

## Базовая настройка Настройка роутеров

Указываем имя и создаем instance
Внимательно MTU 1550 для loopback и 1600 для всех остальных интерфейсов иначе пакеты не дойдут.

### mpls-gw-core

```bash
banner motd ! REASKILLS 2026 !
hostname mpls-gw-core.rea26.ru
ip route 0.0.0.0/0 192.168.122.1
ip pim register-rp-reachability

interface loopback.0
 ip mtu 1550
 ip address 1.1.1.1/32
exit

port ge0
 service-instance TO_BR
  encapsulation untagged
 exit
 service-instance SNMP_SSH
  encapsulation dot1q 100
  rewrite pop 1
  exit
 exit
exit

port ge1
 service-instance TO_CR
  encapsulation untagged
 exit
 service-instance SNMP_SSH
  encapsulation dot1q 100
  rewrite pop 1
  exit
 exit
exit

port ge2
 service-instance TO_INET
  encapsulation untagged
 exit
exit
```

### mpls-gw-br

```bash
hostname mpls-gw-br.rea26.ru
ip pim register-rp-reachability

port ge0
 service-instance TO_HUB_BR
  encapsulation untagged
 service-instance SNMP_SSH
  encapsulation dot1q 100
  rewrite pop 1
  exit
 exit
exit

port ge1
 service-instance TO_BR-CLI
  encapsulation untagged
 service-instance SNMP_SSH
  encapsulation dot1q 100
  rewrite pop 1
  exit
 exit
exit

port ge2
 service-instance TO_CORE
  encapsulation untagged
exit

interface loopback.0
 ip mtu 1550
 ip address 2.2.2.2/32
exit
```

### mpls-gw-cr

```bash
hostname mpls-gw-cr.rea26.ru
ip pim register-rp-reachability

port ge0
 service-instance TO_HUB-CR
  encapsulation untagged
exit

port ge1
 service-instance TO_CR-CLI
  encapsulation untagged
exit

port ge2
 service-instance TO_CORE
  encapsulation untagged
exit

interface loopback.0
 ip mtu 1550
 ip address 3.3.3.3/32
exit
```

## Настройка интерфейсов
### mpls-gw-core

```bash
interface ge0_to_br
 ip mtu 2000
 label-switching
 connect port ge0 service-instance TO_BR
 ip address 10.0.12.1/30
exit

interface ge1_to_cr
 ip mtu 2000
 label-switching
 connect port ge1 service-instance TO_CR
 ip address 10.0.13.1/30
exit
```

### mpls-gw-br

```bash
interface ge2_to_core
 ip mtu 2000
 label-switching
 connect port ge2 service-instance TO_CORE
 ip address 10.0.12.2/30
exit
```

### mpls-gw-cr

```bash
interface ge2_to_core
 ip mtu 2000
 label-switching
 connect port ge2 service-instance TO_CORE
 ip address 10.0.13.2/30
exit
```

## Включаем OSPF

### mpls-gw-core

Также отколючем расылку маршрута в ge2_to_inet

```bash
router ospf 1
 ospf router-id 1.1.1.1
 network 1.1.1.1/32 area 0.0.0.0
 network 10.0.12.0/30 area 0.0.0.0
 network 10.0.13.0/30 area 0.0.0.0
 passive-interface ge2_to_inet
exit
```

### mpls-gw-br

```bash
router ospf 1
 ospf router-id 2.2.2.2
 network 2.2.2.2/32 area 0.0.0.0
 network 10.0.12.0/30 area 0.0.0.0
 network 192.168.100.0/24 area 0.0.0.0
exit
```

### mpls-gw-cr

```bash
router ospf 1
 ospf router-id 3.3.3.3
 network 3.3.3.3/32 area 0.0.0.0
 network 10.0.13.0/30 area 0.0.0.0
 network 192.168.100.0/24 area 0.0.0.0
exit
```


Проверяем что получилось
Проверяем со всех роутеров, в примере данные с core

```bash
show ip ospf neighbor
```

```bash

Total number of full neighbors: 2
OSPF process 1 VRF(default):
Neighbor ID     Pri   State            Dead Time   Address         Interface           Instance ID
2.2.2.2           1   Full/DR          00:00:32    10.0.12.2       ge0_to_br               0
3.3.3.3           1   Full/DR          00:00:36    10.0.13.2       ge1_to_cr               0
```

```bash
show ip route ospf
```

```bash
IP Route Table for VRF "default"
O       2.2.2.2/32 [110/11] via 10.0.12.2, ge0_to_br, 01:15:26
O       3.3.3.3/32 [110/11] via 10.0.13.2, ge1_to_cr, 01:15:17

Gateway of last resort is not set
```

Проверяем пинги соседей с core, должны проходить

```bash
ping 2.2.2.2
```

```bash
PING 2.2.2.2 (2.2.2.2) 56(84) bytes of data.
64 bytes from 2.2.2.2: icmp_seq=1 ttl=64 time=15.6 ms
```

```bash
ping 3.3.3.3
```

```bash
PING 3.3.3.3 (3.3.3.3) 56(84) bytes of data.
64 bytes from 3.3.3.3: icmp_seq=1 ttl=64 time=18.9 ms
```

## Включаем LDP
### mpls-gw-core

```bash
router ldp
 targeted-peer ipv4 2.2.2.2
  exit-targeted-peer-mode
 targeted-peer ipv4 3.3.3.3
  exit-targeted-peer-mode
 transport-address ipv4 1.1.1.1
 ldp label preserve
exit
```

### mpls-gw-br

```bash
router ldp
 targeted-peer ipv4 3.3.3.3
  exit-targeted-peer-mode
 transport-address ipv4 2.2.2.2
exit
```

### mpls-gw-cr

```bash
router ldp
 targeted-peer ipv4 2.2.2.2
  exit-targeted-peer-mode
 transport-address ipv4 3.3.3.3
exit
```

## Включаем LDP 
на loopback на всех роутерах

```bash
interface loopback.0
 ldp enable ipv4
exit
```

### mpls-gw-core

```bash
interface ge0_to_br
 ldp enable ipv4
exit

interface ge1_to_cr
 ldp enable ipv4
exit
```

### mpls-gw-br

```bash
interface ge2_to_core
 ldp enable ipv4
exit
```

### mpls-gw-cr

```bash
interface ge2_to_core
 ldp enable ipv4
exit
```

## Настройка VPLS
Производится только на конечных BR и CR

### mpls-gw-br

```bash
vpls-instance office-lan 100
 vpls-type raw
 member port ge0 service-instance TO_HUB_BR
 member port ge1 service-instance TO_BR-CLI
 signaling ldp
  vpls-peer 3.3.3.3
  exit-signaling
exit
```

### mpls-gw-cr

```bash
vpls-instance office-lan 100
 vpls-type raw
 member port ge0 service-instance TO_HUB-CR
 member port ge1 service-instance TO_CR-CLI
 signaling ldp
  vpls-peer 2.2.2.2
  exit-signaling
exit
```

## Проверка MPLS + LDP

```bash
show mpls ldp neighbor
```

Должны быть соседи по всем интерфейсам

```bash
IP Address                 Intf Name    Holdtime   LDP-Identifier
10.0.12.2                     ge0_to_br   15         2.2.2.2:0
10.0.13.2                     ge1_to_cr   15         3.3.3.3:0
```

```bash
show mpls ldp discovery 
```

Должны быть записи для 1.1.1.1, 2.2.2.2, 3.3.3.3

```bash
ge0_to_br    ge1_to_cr    ge2_to_inet  loopback.0   
mpls-gw-core.rea26.ru#show mpls ldp discovery 
Id      Interface       LDP Identifier          LDP Enabled     Version Merge Capability
6       loopback.0      1.1.1.1:0              Disabled IPv4    Merge capable
7       ge0_to_br       1.1.1.1:0              Enabled  IPv4    Merge capable
8       ge1_to_cr       1.1.1.1:0              Enabled  IPv4    Merge capable
9       ge2_to_inet     1.1.1.1:0              Disabled         N/A
```

```bash
show mpls forwarding-table
```

Должны быть записи типа "Pop" или "Swap"

```bash
Codes: > - installed FTN, * - selected FTN, p - stale FTN,
       B - BGP FTN, K - CLI FTN,
       L - LDP FTN, R - RSVP-TE FTN, S - SNMP FTN, I - IGP-Shortcut,
       U - unknown FTN

Code    FEC                 FTN-ID    Tunnel-id   Pri   LSP-Type        Out-Label    Out-Intf       Nexthop
L>      2.2.2.2/32          1         0           Yes   LSP_DEFAULT     3            ge0_to_br     10.0.12.2
L>      3.3.3.3/32          2         0           Yes   LSP_DEFAULT     3            ge1_to_cr     10.0.13.2
```

```bash
traceroute 2.2.2.2 
```

```bash
traceroute to 2.2.2.2 (2.2.2.2), 30 hops max, 60 byte packets
 1  2.2.2.2 (2.2.2.2)  27.954 ms  27.820 ms  27.735 ms
```


## Включаем BGP
Используем BGP AS 65001 на всех трёх

### mpls-gw-core

```bash
router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 2.2.2.2 remote-as 65001
 neighbor 2.2.2.2 default-originate
 neighbor 3.3.3.3 remote-as 65001
 neighbor 3.3.3.3 default-originate
exit
```

### mpls-gw-br

```bash
router bgp 65001
 bgp router-id 2.2.2.2
 neighbor 1.1.1.1 remote-as 65001
 neighbor 3.3.3.3 remote-as 65001
exit
```

### mpls-gw-cr

```bash
router bgp 65001
 bgp router-id 3.3.3.3
 neighbor 1.1.1.1 remote-as 65001
 neighbor 2.2.2.2 remote-as 65001
exit
```

### Проверка

```bash
show ip bgp summary

```

```bash
BGP router identifier 1.1.1.1, local AS number 65001
BGP table version is 1
0 BGP AS-PATH entries
0 BGP community entries

Neighbor        V    AS     MsgRcv    MsgSen    TblVer  InQ   OutQ   Up/Down   State/PfxRcd
-------------------------------------------------------------------------------------------
2.2.2.2         4    65001  180       180       1       0     0      01:29:02     0
3.3.3.3         4    65001  179       179       1       0     0      01:28:56     0

Total number of neighbors 2

Total number of Established sessions 2
```

```bash
show vpls-instance detail office-lan
```

Состояние peer должно быть Up (не Dn!)

```bash
Virtual Private LAN Service Instance: office-lan, ID: 100
 SIG-Protocol: LDP
 Learning: Enabled
 Group ID: 0
 Configured MTU: 9710
 Description: none
 Operating mode: Raw
 Configured connections:
  Port ge0 Service-instance TO_HUB-CR
  Port ge1 Service-instance TO_CR-CLI
 Mesh Peers:  2.2.2.2 (Up)
mpls-gw-cr.rea26.ru>
```

```bash
show vpls mac-table office-lan
```

После подключения клиентов — должны появиться MAC

```bash
 VPLS Aging time is 60 sec
 
    L2 Address               AC / Peer               Type        VLAN     Age 
 ---------------- ------------------------------- ---------- ----------- -----
  5254.006d.0954  ge0.TO_HUB-CR                   SI                      28     
```

## Выход в «Интернет»

Настроить выход в инернет для клиентов.
  1. Через MPLS пробросить сет 192.168.122.0/24 внутрь и завенуть её в влан, к примеру 5
  2. В каждом офисе на одном из серверов поднять интерфейс который примет этот влан и получит адрес из сети 192.168.122.0/24
  3. Настроить NAT между интерфейсами, использовать этот сервера как шлюз

### mpls-gw-core

```bash
vpls-instance INET 10
 vpls-mtu 9710
 vpls-type raw
  member port ge2 service-instance TO_INET
 signaling ldp
  vpls-peer 2.2.2.2
  vpls-peer 3.3.3.3
  exit-signaling
exit
```

### mpls-gw-br и mpls-gw-cr

Добавлем новый инстанс TO_INET и связываем его в VLAN 5

```bash
port ge0
 service-instance TO_INET
  encapsulation dot1q 5
  rewrite pop 1
 exit
exit

port ge1
 service-instance TO_INET
  encapsulation dot1q 5
  rewrite pop 1
 exit
exit
```

Присоеденится к MPLS и добавить сеть в BGP

```bash
vpls-instance INET 10
 vpls-mtu 9710
 vpls-type raw
 member port ge0 service-instance TO_INET
 member port ge1 service-instance TO_INET
 signaling ldp
  vpls-peer 1.1.1.1
  exit-signaling
exit
```

```bash
router bgp 65001
 network 192.168.1.0/24
exit
```

### server

```bash
/etc/network/interfaces
```

```bash
auto enp1s0.5
iface enp1s0.5 inet dhcp
```

## Далее включаем Формвардин делаем NAT
К примеру настройка на BR-SRV2

```bash
cat /etc/sysctl.conf
net.ipv4.ip_forward=1
```

```bash
sudo sysctl -p
```

```bash
cat /etc/nftables.conf
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0;
    }
    
    chain forward {
        type filter hook forward priority 0;
    }
    
    chain output {
        type filter hook output priority 0;
    }
}

table ip nat {
    chain prerouting {
        type nat hook prerouting priority 0;
        policy accept;
    }
    
    chain postrouting {
        type nat hook postrouting priority 0;
		oifname "eth0.5" ip saddr 192.168.1.0/24 masquerade;
    }
}
```

```bash
sudo systemctl restart nftables.service
```

```bash
cat /etc/network/interfaces
auto lo
iface lo inet loopback

auto enp1s0
iface enp1s0 inet static
        address 192.168.1.14/24
        mtu 1440

auto enp1s0.5
iface enp1s0.5 inet dhcp
        mtu 1440
```

Если TCP трафик не ходит нужно изменить MTU
```bash
ip link set eth0.5 mtu 1440
ip link set eth0 mtu 1440
```


## IP SLA на PE-роутерах (MPLS-GW-BR и MPLS-GW-CR)

```bash
ip sla-profile reaskills
 icmp 192.168.122.103 num-packets 4
 packet-frequency 30
 rtt-threshold 1000
!
```

## Настройка безопастности
Включаем банер.
Разрешаем доступ по SSh и SNMP
Создаем пользователя, отключаем встроеного, включаем шифрование паролей

```bash
banner motd ! REASKILLS 2026 !

security-profile 10
  rule 10 permit udp any any eq 161
  rule 11 permit tcp any any eq 22
exit
security 10

username adminer 
 description sysadmin
 password P@ssw0rd
 role admin
exit
enable password P@ssw0rd
no username admin
service password-encryption
```

## Настройка SNMPv3
Получение данных по SNMPv3 с CR, но по задани нужно на всех

Включем задаем паольь и права, если не сделать view то ничего не отдаст

```bash
snmp-server enable snmp 
snmp-server view view1 .1 included
snmp-server group reaskills v3 auth read view1
```

Далее нужно настроить порт для управления, используем VLAN 100
Не дает создать interface с mgmt в имени!!! Пишет используется зарезрвирование системное имя.

```bash
port ge0
 service-instance SNMP_SSH
  encapsulation dot1q 100
  rewrite pop 1
 exit
exit

interface snmp_ssh
 ip mtu 1500
 connect port ge0 service-instance SNMP_SSH
 ip address 192.168.100.3/24
exit

router ospf 1
 network 192.168.100.0/24 area 0.0.0.0
exit
```

На Компе, нужно давить влан чтобы ходить к роутеру

```bash
/etc/network/interfaces

auto eth0.100
iface eth0.100 inet static
        address 192.168.100.13/24
```

Добавить маршрутизацию чтобы ходить через VLAN 100 можно было ходить на loopback адреса 

```bash
ip route add 1.1.1.1/32 via 192.168.100.3
ip route add 2.2.2.2/32 via 192.168.100.3
ip route add 3.3.3.3/32 via 192.168.100.3
```

## TFTP Backup
Автоматизировать скидывание конфигов на TFTP сервер, развернуть сервер TFTP

Ставим это сервер, там есть другие но этот проще настривать

```bash
sudo apt install tftpd-hpa
```

Папка куда будут падать конфиги

```bash
sudo mkdir -p /opt/configs
sudo chown nobody:nogroup /opt/configs
sudo chmod 755 /opt/configs
```

настройка сервера TFTP

```bash
sudo nano /etc/default/tftpd-hpa
TFTP_USERNAME="nobody"
TFTP_DIRECTORY="/opt/configs"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="--secure --create"
```

Не забыть открыть порт!!!

```bash
sudo ufw allow 69/udp
sudo ufw reload
```

и проверить что он откыт

```bash
sudo ss -uln | grep :69
```

```bash
UNCONN 0 0 *:69 *:*
```

Перезапустить и проверить что служба включена!

```bash
sudo systemctl restart tftpd-hpa
sudo systemctl status tftpd-hpa
```

Можно скинуть конфиг с коммутатора используем ранее натсроеный VLAN 100

```bash
copy startup-config tftp tftp://192.168.100.13/mpls-gw-cr.rea26.ru.cfg
```
