[Конкурсное задание компетенции “Сетевое и системное администрирование” чемпионата ReaSkills 2026](https://gitflic.ru/project/alts25/skills-itnsa)

Я начал настройку с **ISP-SRV**, на нем должно быть:

 - Репозиторий
 - DNS через bind9

Устанавливаем имя и сразу редактируем приглашение (подсветка алиасы и проче что хочется можно включить раскаментировав соотвествующие разделы):

    hostnamectl set-hostname isp-srv.rea26.skills
    nano /root/.bashrc /etc/bash.bashrc
    exec bash

сразу меняем debian на имя пк в **/etc/hosts**

    127.0.0.1       isp-srv.rea26.skills isp-srv

Настройка репозиториев, заменяем ***.debina.org** на **mirror.yandex.ru** в **/etc/apt/sources.list**

Далее добавлем в **/etc/apt/apt.conf.d/90-proxy.conf**

    Acquire::http::Proxy "http://proxy.tech.skills:3128";
    Acquire::https::Proxy "http://proxy.tech.skills:3128";

Далее нужно добавить в **/etc/dhcpcd.conf** в конец строку, чтобы разрешались имена **tech.skills**

    static domain_name_servers=10.113.38.139 192.168.122.1

И рестартим службу чтобы всё заработало

    systemctl restart networking.service

Устанавливаем софт

    apt update
    apt install bind9 curl
Пакет **wget** **bind9-host** уже установлены


Продолжаем с **OUT-CLI**

Настройка имени:

    hostnamectl set-hostname out-cli.rea26.skills
    nano /root/.bashrc /etc/bash.bashrc
    exec bash

сразу меняем **astra** на имя пк в **/etc/hosts**

    127.0.0.1      out-cli.rea26.skills out-cli

Настройка репозиториев, заменяем **stable** на **frozen** и указываем версию **1.8.1** в **/etc/apt/sources.list**

    deb https://dl.astralinux.ru/astra/frozen/1.8_x86-64/1.8.1/main-repository/     1.8_x86-64 main contrib non-free non-free-firmware
    deb https://dl.astralinux.ru/astra/frozen/1.8_x86-64/1.8.1/extended-repository/ 1.8_x86-64 main contrib non-free non-free-firmware

Далее добавлем в **/etc/apt/apt.conf.d/90-proxy.conf**

    Acquire::http::Proxy "http://proxy.tech.skills:3128";
    Acquire::https::Proxy "http://proxy.tech.skills:3128";

и нужно в графике через NetworkManager добавить дополнительный DNS сервер 10.113.38.139, заем презапускаем подключения.

Проверяем что инет у нас появился и репозитрии подтянулись
    apt update
    apt install curl wget bind9-host
