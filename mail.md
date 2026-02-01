# Инструкция по настройке почтового сервера на CR-SRV

## Шаг 1: Установка необходимого ПО

```bash
# Обновление списка пакетов и установка почтового стека
sudo apt update
sudo apt install postfix dovecot-imapd dovecot-gssapi -y
```

> **При установке `postfix` выберите:**
> - Тип конфигурации: **«Интернет-сайт»**
> - Имя системы: **`rea26.skills`**

## Использовать `dpkg-reconfigure postfix`

Ответы при интерактивной настройке:

| Вопрос | Ответ |
|--------|-------|
| **Тип конфигурации** | `2. Интернет-сайт` |
| **Имя системы** | `rea26.skills` |
| **Получатель почты для root** | (пусто) |
| **Другие адреса для приёма почты** | `rea26.skills, mail.rea26.skills, localhost.rea26.skills, localhost` |
| **Принудительно задействовать синхронные обновления** | `Да` |
| **Локальные сети** | `127.0.0.0/8, 192.168.1.0/24, 192.168.122.0/24` |
| **Ограничение размера почтового ящика** | `0` (без ограничений) |
| **Символ расширения локальных адресов** | `+` |
| **Использовать протокол** | `3. Только IPv4` |

---

## Шаг 2: Настройка доверия корпоративному ЦС

```bash
# Копирование корневого сертификата в системное хранилище
sudo cp /etc/ca/ca.crt /usr/local/share/ca-certificates/rea26-ca.crt
sudo update-ca-certificates
```

> Сертификаты для почтового сервера должны находиться в `/etc/ca/`:
> - `/etc/ca/issued/cr-srv.rea26.skills.crt` — сертификат сервера
> - `/etc/ca/private/cr-srv.rea26.skills.key` — закрытый ключ
> - `/etc/ca/ca.crt` — корневой сертификат ЦС

---

## Шаг 3: Настройка Postfix (`/etc/postfix/main.cf`)

```bash
sudo nano /etc/postfix/main.cf
```

Добавьте/измените следующие параметры:

```ini
# === Основные параметры ===
myhostname = cr-srv.rea26.skills
mydomain = rea26.skills
myorigin = $mydomain
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
mynetworks = 127.0.0.0/8, 192.168.1.0/24, 192.168.122.0/24

# === TLS ===
smtpd_tls_cert_file = /etc/ca/issued/cr-srv.rea26.skills.crt
smtpd_tls_key_file = /etc/ca/private/cr-srv.rea26.skills.key
smtpd_tls_CAfile = /etc/ca/ca.crt
smtpd_use_tls = yes
smtpd_tls_security_level = may
smtpd_tls_auth_only = yes

# === SASL через Dovecot ===
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_relay_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination # Проверить?

# === Формат почтовых ящиков ===
home_mailbox = Maildir/
mail_spool_directory = /var/mail

# === Ограничения ===
message_size_limit = 52428800
line_length_limit = 3072
```

---

## Шаг 7: Настройка Dovecot — аутентификация через GSSAPI

### 7.1. Включить GSSAPI в `/etc/dovecot/conf.d/10-auth.conf`

```bash
sudo nano /etc/dovecot/conf.d/10-auth.conf
```

```ini
disable_plaintext_auth = no
auth_mechanisms = plain login
```

### 7.3. Настроить TLS (`/etc/dovecot/conf.d/10-ssl.conf`)

```ini
ssl = required
ssl_cert = </etc/ca/issued/cr-srv.rea26.skills.crt
ssl_key = </etc/ca/private/cr-srv.rea26.skills.key
ssl_ca = </etc/ca/ca.crt
```

### 7.4. Настроить сокет аутентификации для Postfix (`/etc/dovecot/conf.d/10-master.conf`)

Найдите секцию `service auth` и добавьте:

```ini
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }
}
```

### 7.5. Указать формат почтовых ящиков (`/etc/dovecot/conf.d/10-mail.conf`)

```ini
mail_location = maildir:~/Maildir
```

---

## Шаг 8: Настройка DNS — MX запись

На сервере `ISP-SRV` (BIND) добавьте в зону `rea26.skills`:

```dns
rea26.skills.    IN  MX  10  mail.rea26.skills.
mail             IN  A   <IP_адрес_CR-SRV>
```

> Замените `<IP_адрес_CR-SRV>` на реальный IP адрес сервера (например, `192.168.1.13`)

---

## Шаг 9: Перезапуск служб

```bash
sudo systemctl restart postfix dovecot
sudo systemctl enable postfix dovecot
```

---

## Шаг 10: Проверка работоспособности

### 10.1. Проверка получения почты

```bash
# Отправка тестового письма локальному пользователю
echo "Test message from Postfix" | mail -s "Test IMAP" eva@rea26.skills
```

### 10.2. Проверка аутентификации Kerberos

```bash
# Получение билета для тестового пользователя
kinit lori@REA26.SKILLS
klist  # Просмотр полученного билета
```

### 10.3. Проверка логов

```bash
# Просмотр логов в реальном времени
sudo tail -f /var/log/mail.log
```

### 10.4. Тестирование с почтового клиента

На клиенте `CR-CLI`:
1. Откройте Thunderbird / Outlook
2. Укажите адрес: `lori@rea26.skills`
3. Введите пароль учётной записи домена
4. Клиент автоматически определит параметры через автонастройку
5. Отправьте письмо на `eva@rea26.skills` (BR-CLI)
6. Убедитесь, что письмо доставлено без запроса дополнительных параметров

---

## Схема аутентификации

```
Почтовый клиент (Thunderbird)
         │
         ├── IMAP (порт 993) ──► Dovecot ──► GSSAPI ──► FreeIPA (Kerberos)
         │
         └── SMTP (порт 587) ──► Postfix ──► SASL/Dovecot ──► GSSAPI ──► FreeIPA
```
