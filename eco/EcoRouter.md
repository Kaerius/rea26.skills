# Инструкция по настройке сетевой инфраструктуры на EcoRouter OS
## ReaSkills 2026 — Модуль C: Пуско-наладка сетевой инфраструктуры
> **Важно:**
> Все конфигурации ниже соответствуют финальным рабочим конфигам и проверены в топологии задания. Последовательность шагов оптимизирована для корректной работы MPLS/LDP/VPLS.
> BGP - не настаиваем! Это был артефакт прошлых версий.
> MTU на VPLS ведущий в интернет 1500, остальные MTU оставляем по умолчанию. MTU на клиентах не трогаем!
> Сеть 192.168.100.0/24 предназначена для доступа по SSH и SNMP, вход через VLAN 100 от CR-SRV, он будет и шлюзом для остальных в эту сеть. Нужно будет прописать маршруты руками на CR-SRV. И добавить записи в DNS!
> Адреса можно вписать свои, главное не запутаться!

Копи паст конфиги: [MPLS-GW-CORE](./eco/MPLS-GW-CORE.cfg) [MPLS-GW-BR](./eco/MPLS-GW-BR.cfg) [MPLS-GW-CR](./eco/MPLS-GW-CR.cfg) 

Конфиги с устройств: [MPLS-GW-CORE](./eco/mpls-gw-core.cfg) [MPLS-GW-BR](./eco/mpls-gw-br.cfg) [MPLS-GW-CR](./eco/mpls-gw-cr.cfg) 

---

## 📋 Топология и адресация

| Устройство | Loopback | Интерфейс к CORE | Интерфейс к клиентам | Роль |
|------------|----------|------------------|----------------------|------|
| **MPLS-GW-CORE** | 1.1.1.1/32 | — | ge0: 10.0.12.1/30 (к BR)<br>ge1: 10.0.13.1/30 (к CR)<br>ge2: 192.168.122.10/24 (Интернет) | P-устройство |
| **MPLS-GW-BR** | 2.2.2.2/32 | ge2: 10.0.12.2/30 | ge0: untagged (офис)<br>ge1: untagged (клиенты) | PE-устройство |
| **MPLS-GW-CR** | 3.3.3.3/32 | ge2: 10.0.13.2/30 | ge0: untagged (офис) + VLAN 100 (управление)<br>ge1: untagged (клиенты) | PE-устройство |

---

## 🔧 Базовая конфигурация (все устройства)

```bash
# Установка имени и баннера
banner motd ! REASKILLS 2026 !
hostname <hostname>.rea26.ru  # Например: mpls-gw-br.rea26.ru

# Учетные записи и безопасность
username adminer
 description sysadmin
 password P@ssw0rd
 role admin
!
enable password P@ssw0rd
no username admin
service password-encryption

# Профиль безопасности (разрешить SSH и SNMP)
security-profile 10
 rule 10 permit udp any any eq 161
 rule 11 permit tcp any any eq 22
!
security 10
```

> **Проверка:**  
> `show users localdb` — убедиться, что активна только учетная запись `adminer`  
> `show running-config | include password` — пароли должны отображаться в зашифрованном виде

---

## 🌐 Настройка физических портов и сервис-инстансов

### MPLS-GW-CORE
```bash
port ge0
 service-instance TO_BR
  encapsulation untagged
 exit
exit

port ge1
 service-instance TO_CR
  encapsulation untagged
 exit
exit

port ge2
 service-instance TO_INET
  encapsulation untagged
 exit
exit
```

### MPLS-GW-BR
```bash
port ge0
 service-instance TO_HUB_BR
  encapsulation untagged
 exit
 service-instance TO_INET
  encapsulation dot1q 5
  rewrite pop 1
 exit
exit

port ge1
 service-instance TO_BR_CLI
  encapsulation untagged
 exit
 service-instance TO_INET
  encapsulation dot1q 5
  rewrite pop 1
 exit
exit

port ge2
 service-instance TO_CORE
  encapsulation untagged
 exit
exit
```

### MPLS-GW-CR
```bash
port ge0
 service-instance TO_HUB_CR
  encapsulation untagged
 exit
 service-instance TO_INET
  encapsulation dot1q 5
  rewrite pop 1
 exit
 service-instance SNMP_SSH
  encapsulation dot1q 100
  rewrite pop 1
 exit
exit

port ge1
 service-instance TO_CR_CLI
  encapsulation untagged
 exit
 service-instance TO_INET
  encapsulation dot1q 5
  rewrite pop 1
 exit
exit

port ge2
 service-instance TO_CORE
  encapsulation untagged
 exit
exit
```

> **Важно:**  
> - На `MPLS-GW-BR` отсутствует сервис-инстанс `SNMP_SSH` — управление осуществляется через клиентские сети  
> - На `MPLS-GW-CR` интерфейс управления вынесен в отдельный сервис-инстанс `SNMP_SSH` (VLAN 100)

---

## 🔌 Настройка логических интерфейсов

### Все устройства — Loopback
```bash
interface loopback.0
 ip address <loopback-ip>/32  # 1.1.1.1 / 2.2.2.2 / 3.3.3.3
 ldp enable ipv4
exit
```

### MPLS-GW-CORE — Интерфейсы к другим роутерам
```bash
interface ge0-to-br
 label-switching
 connect port ge0 service-instance TO_BR
 ip address 10.0.12.1/30
 ldp enable ipv4
exit

interface ge1-to-cr
 label-switching
 connect port ge1 service-instance TO_CR
 ip address 10.0.13.1/30
 ldp enable ipv4
exit

interface ge2_to_inet
 ip address 192.168.122.10/24
exit

# Статический маршрут по умолчанию к шлюзу провайдера
ip route 0.0.0.0/0 192.168.122.1
```

### MPLS-GW-BR — Интерфейс к CORE
```bash
interface ge2-to-core
 label-switching
 connect port ge2 service-instance TO_CORE
 ip address 10.0.12.2/30
 ldp enable ipv4
exit
```

### MPLS-GW-CR — Интерфейсы к CORE и управлению
```bash
interface ge2-to-core
 ip mtu 2000
 label-switching
 connect port ge2 service-instance TO_CORE
 ip address 10.0.13.2/30
 ldp enable ipv4
exit

interface snmp-ssh
 connect port ge0 service-instance SNMP_SSH
 ip address 192.168.100.3/24
exit
```

> **Критично:**  
> - Внешний интерфейс (`ge2_to_inet`) **не** должен иметь `label-switching`

---

## 📡 Настройка OSPF

### MPLS-GW-CORE
```bash
router ospf 1
 ospf router-id 1.1.1.1
 network 1.1.1.1/32 area 0.0.0.0
 network 10.0.12.0/30 area 0.0.0.0
 network 10.0.13.0/30 area 0.0.0.0
 network 192.168.100.1/32 area 0.0.0.0
exit
```

### MPLS-GW-BR
```bash
router ospf 1
 ospf router-id 2.2.2.2
 network 2.2.2.2/32 area 0.0.0.0
 network 10.0.12.0/30 area 0.0.0.0
 network 192.168.100.2/32 area 0.0.0.0
exit
```

### MPLS-GW-CR
```bash
router ospf 1
 ospf router-id 3.3.3.3
 network 3.3.3.3/32 area 0.0.0.0
 network 10.0.13.0/30 area 0.0.0.0
 network 192.168.100.3/32 area 0.0.0.0
exit
```

> **Проверка:**  
> `show ip ospf neighbor` — должно быть 2 соседа на CORE, по 1 на каждом PE  
> `show ip route ospf` — должны отображаться маршруты к loopback всех роутеров

---

## 🏷️ Настройка LDP

### MPLS-GW-CORE
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

### MPLS-GW-BR
```bash
router ldp
 targeted-peer ipv4 3.3.3.3
  exit-targeted-peer-mode
 transport-address ipv4 2.2.2.2
exit
```

### MPLS-GW-CR
```bash
router ldp
 targeted-peer ipv4 2.2.2.2
  exit-targeted-peer-mode
 transport-address ipv4 3.3.3.3
exit
```

> **Проверка:**  
> `show mpls ldp neighbor` — должны отображаться все соседи  
> `show mpls forwarding-table` — должны присутствовать записи с метками (Pop/Swap)

---

## 🌐 Настройка VPLS

### VPLS office-lan (L2-связность офисов, ID 100)

#### MPLS-GW-BR
```bash
vpls-instance office-lan 100
 member port ge0 service-instance TO_HUB_BR
 member port ge1 service-instance TO_BR_CLI
 signaling ldp
  vpls-peer 3.3.3.3
  exit-signaling
exit
```

#### MPLS-GW-CR
```bash
vpls-instance office-lan 100
 member port ge0 service-instance TO_HUB_CR
 member port ge1 service-instance TO_CR_CLI
 signaling ldp
  vpls-peer 2.2.2.2
  exit-signaling
exit
```

### VPLS INET (доступ в Интернет через VLAN 5, ID 10)

#### MPLS-GW-CORE
```bash
vpls-instance inet-lan 10
 vpls-mtu 1500
 member port ge2 service-instance TO_INET
 signaling ldp
  vpls-peer 2.2.2.2
  vpls-peer 3.3.3.3
  exit-signaling
exit
```

#### MPLS-GW-BR и MPLS-GW-CR
```bash
vpls-instance inet-lan 10
 vpls-mtu 1500
 member port ge0 service-instance TO_INET
 member port ge1 service-instance TO_INET
 signaling ldp
  vpls-peer 1.1.1.1
  exit-signaling
exit
```

> **Проверка:**  
> `show vpls-instance detail office-lan` — статус пира должен быть `Up`, проверяем на BR/CR
> `show vpls mac-table office-lan` — после подключения клиентов должны отображаться MAC-адреса, проверяем на BR/CR
> `show vpls-instance detail inet-lan` — статус пира должен быть `Up`, проверяем на CORE/BR/CR
> `show vpls mac-table inet-lan` — после подключения клиентов должны отображаться MAC-адреса, проверяем на CORE/BR/CR

---

## 🛡️ Мониторинг и управление

### IP SLA (на PE-устройствах: BR и CR)
```bash
ip sla-profile reaskills
 icmp 192.168.122.103 num-packets 4
 packet-frequency 30
 rtt-threshold 1000
exit
```

### SNMPv3 (все устройства)
```bash
snmp-server enable snmp
snmp-server view view1 .1 included
snmp-server group reaskills v3 auth read view1
snmp-server user snmpuser group reaskills auth md5 snmppass
```

> **Доступ по SNMP:**  
> - Группа: `reaskills`  
> - Пользователь: `snmpuser` / Пароль: `snmppass` (настраивается на стороне клиента)  
> - Доступ разрешен только из сети управления (VLAN 100)

---

## ✅ Финальная проверка

| Проверка | Команда | Ожидаемый результат |
|----------|---------|---------------------|
| Соседи OSPF | `show ip ospf neighbor` | Все соседи в состоянии `Full` |
| Соседи LDP | `show mpls ldp neighbor` | Все соседи присутствуют |
| Состояние VPLS | `show vpls-instance detail office-lan` | Пиры в состоянии `Up` |
| Маршруты по умолчанию | `show ip route 0.0.0.0/0` | Маршрут присутствует на PE |
| Доступ в Интернет | `ping 8.8.8.8 source 192.168.1.1` | Успешный пинг с клиентского хоста |
| Доступ между офисами | `ping 192.168.1.20` (из другого офиса) | Успешный пинг (один широковещательный домен) |

---

## ⚠️ Критические замечания

1. **MTU:**  
   - VPLS в интернте: `vpls-mtu 1500`  
   Нарушение этих значений приведет к потере пакетов.

2. **Безопасность управления:**  
   - Доступ по SSH к офисным роутерам разрешен **только** из сети управления (VLAN 100) и дальнейшем доступ через шлюз.

3. **Резервное копирование:**  
   Для выгрузки конфигурации на TFTP-сервер (`192.168.100.13`):
   ```bash
   copy startup-config tftp tftp://192.168.100.13/<hostname>.cfg
   ```

4. **Перезагрузка:**  
   Перед сдачей работы выполните `reload` на всех маршрутизаторах и убедитесь, что вся функциональность восстанавливается автоматически.

---

> Инструкция составлена на основе финальных рабочих конфигураций и соответствует требованиям задания ReaSkills 2026. Все параметры проверены в рабочей топологии.
