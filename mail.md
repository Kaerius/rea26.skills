# Инструкция по настройке почтового сервера на CR-SRV
## Шаг 0: Установка необходимого ПО

Делаем клиентом Домена! Чтобы доменые УЗ былыи локальными и не настривать ldap соедениение.

## Шаг 1: Установка необходимого ПО

```bash
# Обновление списка пакетов и установка почтового стека
sudo apt update
sudo apt install postfix dovecot-imapd
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
| **Использовать протокол** | `3` (Только IPv4) |

---

## Шаг 2: Настройка доверия корпоративному ЦС

```bash
# Копирование корневого сертификата в системное хранилище
sudo cp /etc/ca/ca.crt /usr/local/share/ca-certificates/rea26-ca.crt
sudo update-ca-certificates
```

Выдать права на сетрификаты (Добавить правильные права или скопировать серт)
```bash
sudo chmod 777 -R /etc/ca
```

> Сертификаты для почтового сервера должны находиться в `/etc/ca/`:
> - `/etc/ca/issued/rea26.crt` — сертификат сервера
> - `/etc/ca/private/rea26.key` — закрытый ключ
> - `/etc/ca/ca.crt` — корневой сертификат ЦС

---

## Шаг 3: Настройка конфигурационных файлов

### **1. Файл: `conf.d/10-master.conf`**

- Раскоментируем строки с портами и SSL
- Раскоментируем раздел unix_listener /var/spool/postfix/private/auth идобавлем Пользователя и группу postfix 

```ini
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

unix_listener /var/spool/postfix/private/auth {
  mode = 0666
  user = postfix
  group = postfix
}
```

---

### **2. Файл: `conf.d/10-auth.conf`**

- Отключена опция disable_plaintext_auth (изменено с "yes" на "no")
- Изменен механизм аутентификации с "plain" на "plain login"
- Проверяем что раскомментированы параметры для аутентификации с использованием SASL

```ini
disable_plaintext_auth = no
auth_mechanisms = plain login
```

---

### **3. Файл: `main.cf`**

- Обновлены TLS-параметры: заменены сертификаты snakeoil на rea26
- Добавлен параметр mail_spool_directory = /var/maildir/
- Установлен line_length_limit = 3072
- Настроена интеграция с Dovecot: sasl_type = dovecot, sasl_auth_enable = yes параматры и из значения можно посмотреть в мане `man 5 postconf`

```ini
smtpd_tls_cert_file = /etc/ca/issued/rea26.crt
smtpd_tls_key_file = /etc/ca/private/rea26.key
smtpd_use_tls = yes
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache

mail_spool_directory = /var/maildir/
line_length_limit = 3072

smtpd_sasl_type = dovecot
smtpd_sasl_auth_enable = yes
smtpd_sasl_path = private/auth
```

---

### **4. Файл: `conf.d/10-ssl.conf`**

- Обновлены SSL-сертификаты для использования rea26 вместо snakeoil
- Добавлен namespace для "Sent Messages" (отправленных сообщений)


```ini
ssl_cert = </etc/ca/issued/rea26.crt
ssl_key = </etc/ca/private/rea26.key
```

---

### **5. Файл: `conf.d/15-mailboxes.conf`**

Добавлем auto = subscribe

```ini
namespace inbox {
  mailbox "Sent Messages" {
    special_use = \Sent
    auto = subscribe
  }
}
```

## Шаг 4: Настройка DNS — MX запись

На сервере `CR-DC` (BIND) добавьте в зону `rea26.skills` запись:

```dns
rea26.skills.    IN  MX  10  mail.rea26.skills.
mail             IN  A   <IP_адрес_CR-SRV>
```

> Замените `<IP_адрес_CR-SRV>` на реальный IP адрес сервера (например, `192.168.1.1`)

---

## Шаг 5: Перезапуск служб

```bash
sudo systemctl restart postfix dovecot
sudo systemctl enable postfix dovecot
```

---

## Шаг 6: Проверка работоспособности


### 6.1. Проверка конфигов

```bash
postconf
```

```bash
dovconf
```

### 6.2. Проверка получения почты

Смотрим журнал на предмет ошибок

```bash
journalctl -u dovecot
journalctl -u postfix
```

### 6.3. Проверка логов
Авторизуемся в граике и настривам почу. 

Проверяем путем отправки письма самаому себе.

Не должно быть не ошибок не предупреждений, настройки должны подтянутся автоматом.
