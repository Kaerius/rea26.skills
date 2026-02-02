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
