# Настройка EcoRouter

[Сохраненный конфиг с роутеров](https://pastebin.com/h52ivJCx)

## Базовая настройка Настройка роутеров

Указываем имя и создаем instance

### mpls-gw-core
```bash
hostname mpls-gw-core.rea26.ru
ip route 0.0.0.0/0 192.168.122.1
ip pim register-rp-reachability

interface loopback.0
 ip mtu 1500
 ip address 1.1.1.1/32
exit

port ge0
 mtu 9728
 service-instance TO_BR
  encapsulation untagged
 exit
exit

port ge1
 mtu 9728
 service-instance TO_CR
  encapsulation untagged
 exit
exit

port ge2
 mtu 9728
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
 mtu 9728
 service-instance TO_HUB_BR
  encapsulation untagged
exit

port ge1
 mtu 9728
 service-instance TO_BR-CLI
  encapsulation untagged
exit

port ge2
 mtu 9728
 service-instance TO_CORE
  encapsulation untagged
exit

interface loopback.0
 ip mtu 1500
 ip address 2.2.2.2/32
exit
```

### mpls-gw-cr
```bash
hostname mpls-gw-cr.rea26.ru
ip pim register-rp-reachability

port ge0
 mtu 9728
 service-instance TO_HUB
  encapsulation untagged
exit

port ge1
 mtu 9728
 service-instance TO_CR-CLI
  encapsulation untagged
exit

port ge2
 mtu 9728
 service-instance TO_CORE
  encapsulation untagged
exit

interface loopback.0
 ip mtu 1500
 ip address 3.3.3.3/32
exit
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

## Включаем LDP на loopback на всех роутерах
```bash
interface loopback.0
 ldp enable ipv4
exit
```

## Настройка интерфейсов
### mpls-gw-core
```bash
interface ge0_to_br
 ip mtu 1500
 label-switching
 connect port ge0 service-instance TO_BR
 ip address 10.0.12.1/30
 ldp enable ipv4
exit

interface ge1_to_cr
 ip mtu 1500
 label-switching
 connect port ge1 service-instance TO_CR
 ip address 10.0.13.1/30
 ldp enable ipv4
exit

interface ge2_to_inet
 ip mtu 1500
 connect port ge2 service-instance TO_INET
 ip address 192.168.122.10/24
exit
```

### mpls-gw-br
```bash
interface ge2_to_core
 ip mtu 1500
 label-switching
 connect port ge2 service-instance TO_CORE
 ip address 10.0.12.2/30
 ldp enable ipv4
exit
```

### mpls-gw-cr
```bash
interface ge2_to_core
 ip mtu 1500
 label-switching
 connect port ge2 service-instance TO_CORE
 ip address 10.0.13.2/30
 ldp enable ipv4
exit
```

## Включаем OSPF
### mpls-gw-core
```bash
router ospf 1
 ospf router-id 1.1.1.1
 network 1.1.1.1/32 area 0.0.0.0
 network 10.0.12.0/30 area 0.0.0.0
 network 10.0.13.0/30 area 0.0.0.0
exit
```

### mpls-gw-br
```bash
router ospf 1
 ospf router-id 2.2.2.2
 network 2.2.2.2/32 area 0.0.0.0
 network 10.0.12.0/30 area 0.0.0.0
exit
```

### mpls-gw-cr
```bash
router ospf 1
 ospf router-id 3.3.3.3
 network 3.3.3.3/32 area 0.0.0.0
 network 10.0.13.0/30 area 0.0.0.0
exit
```







## Включаем BGP
используем 65001

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

## Настройка VPLS
Производится только на конечных BR b CR

### mpls-gw-br
```bash
vpls-instance office-lan 100
 vpls-mtu 9710
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
 vpls-mtu 9710
 vpls-type raw
 member port ge0 service-instance TO_HUB
 member port ge1 service-instance TO_CR-CLI
 signaling ldp
  vpls-peer 2.2.2.2
  exit-signaling
exit
```






