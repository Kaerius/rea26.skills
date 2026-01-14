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
