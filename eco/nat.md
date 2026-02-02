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
