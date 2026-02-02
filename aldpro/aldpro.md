# Установка AldPro на CR-DC
- Настройка сети

```bash
cat /etc/network/interfaces
```
```bash
auto lo
iface lo inet loopback

auto eth0
iface etho0 inet static
	address 192.168.1.11/24
	gateway 192.168.1.14
	mtu 1440
```

```bash
sudo systemctl restart networking.service
```
- Настраиваем хостнейм и hosts-файл

```bash
sudo hostnamectl set-hostname cr-dc.rea26.skills
exec bash
```

```bash
cat /etc/hosts
```

```bash
127.0.0.1       localhost
192.168.1.11    cr-dc.rea26.skills cr-dc
```

- Прописываем dns в /etc/resolv.conf:

```bash
cat /etc/resolv.conf
```

```bash
namserver 10.113.38.139
```
- Подключаем репозитории СТРОГО frozen и aldpro (их тоже frozen)

```bash
cat /etc/apt/sources.list
```

```bash
deb https://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.7/repository-base/     1.7_x86-64 main contrib non-free
deb https://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.7/repository-extended/ 1.7_x86-64 main contrib non-free
deb https://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.7/repository-main      1.7_x86-64 main non-free contrib
deb https://dl.astralinux.ru/astra/frozen/1.7_x86-64/1.7.7/repository-update    1.7_x86-64 main contrib non-free
deb https://dl.astralinux.ru/aldpro/frozen/01/3.0.0/                            1.7_x86-64 main base
```
- Провеяе что система обновлена до 1.7.7

```bash
cat /etc/astra/build_version
```

```bash
sudo astra-modeswitch getname
sudo astra-update -A -r
```
- Переводим систему в режим "Смоленск" (это обязательно):

```bash
sudo astra-modeswitch set 2
sudo astra-modeswitch getname
```
- В обновленной системе устанавливаем ALD Pro с Syncer, с GC, но не настраиваем GC

```bash
sudo apt install aldpro-mp aldpro-syncer aldpro-gc
```
- После установки перезагружаем хост

```bash
reboot
```
- Настраиваем ALD Pro СТРОГО без GC

```bash
sudo aldpro-server-install -d rea26.skills -n cr-dc -p P@ssw0rd -u admin --ip 192.168.1.11 --setup_syncer
```
- После установки перезагружаем хост

```bash
reboot
```
- Настраиваем GC:

```bash
aldpro-gc --action install
```
- Проверяем, что всё работает

```bash
aldproctl status
```
- В /etc/bind/ip-options-ext.conf отключаем dnssec и разрешаем рекурсивные запросы:
- 
```bash
cat /etc/bind/ip-options-ext.conf
```

```bash
allow-recursion { any; };
dnssec-validation no;
```
- Перезапуск bind
- 
```bash
cat /etc/bind/ip-options-ext.conf
```

```bash
sudo systemctl restart bind9-pkcs11.service bind9-resolvconf.service
```
# Далее настраиваем клиент:
- Аналогично выполняем базовую настройку (имя хоста, репозитории для 1.8.1 frozen, сеть - здесь допускается оставлять Network Manager и работать через него, в качестве dns указывается КД)
- Устанавливаем пакет aldpro-client и вводим хост в домен
