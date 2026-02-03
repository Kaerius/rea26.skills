# Настройка отправки логов с коммутаторов

```bash
apt install rsyslog
```

## В файле `/etc/rsyslog.conf` раскоментируем раздел

```bash
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
```


## Создать правило приема логов

К примеру файл `/etc/rsyslog.d/remote-logs.conf` со следующем содержимиым

```bash
$template RemLogs, "/opt/logs/%HOSTNAME%.log"
*.* ?RemLogs
```

---


## Добавим правло ротации логов

`/etc/logrotate.d/remote-logs`

```bash
/opt/logs/*.log {
    size 10M
    compress
    copytruncate
    missingok
    notifempty
}
```

### Редактируем logrotate.timer командой systemctl edit logrotate.timer (подсмотерть можно выполнив команду systemctl --full edit logrotate.timer)
```
[Timer]
#OnCalendar=*-*-* *:*00
OnCalendar=*:0/1
AccuracySec=1s
```
### Перезапускаем службы logrotate rsyslog
```
systamctl restart logrotate rsyslog
```



## Включить передачу логов с роутера
```bash
rsyslog host 192.168.100.13 vr default
```




# 3.5 Аккаунтинг (Syslog) (стр.38 ER_UserGuide.pdf 2018)

Функции аутентификации осуществляются при помощи создания пользователей в локальной базе данных.

Функции авторизации реализуются путем привязки к пользователю конкретной роли с определенным набором команд, который может быть изменен по усмотрению пользователя.

Функции аккаунтинга реализованы через отправку лог-данных на удаленный сервер с помощью встроенных в маршрутизатор функций отправки сообщений стандарта Syslog (rsyslog). 

Команда настройки отправки Syslog-сообщений выглядит следующим образом:

`rsyslog host <address> {mgmt | vr {default | <VR_NAME>}}`

Где address - это IP-адрес сервера, на который будут отправляться логи. 

В свою очередь, сообщения могут отправляться через management-порт mgmt или виртуальный маршрутизатор vr {default | <VR_NAME>}, где параметр VR_NAME является именем виртуального маршрутизатора, а default подразумевает стандартный (невиртуализированный) маршрутизатор.



