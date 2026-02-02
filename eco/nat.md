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
        mtu 1440## TFTP Backup
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
