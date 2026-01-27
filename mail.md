# Настройка почтового скервера на сервере CR-SRV

Должен быть в домене?

После развертывения центра сертификации?

Ставим софт
```bash
apt install dovecot-imapd postfix dovecot-gssapi -y
```

Отвечаем: 2. Интернет-сайт

Указываем имя: office.rea26.skills

Запускам конфиг сервера:

```bash
dpkg-reconfigure postfix
```

- Отвечаем: 2. Интернет-сайт
- Указываем имя: office.rea26.skills
- Получатель почты для root и postmaster: (пусто)
- Другие адреса, для которых принимать почту: BR-CLI.rea62.skills, CR-CLI.rea62.skills, CR-SRV.rea62.skills, localhost, mail.office.rea26.skills, office.rea26.skills (указываем все адреса)
- Принудительно задействовать синхроные обновления почтовой очереди?: Да
- Локальные сети: 127.0.0.0/8, 192.168.1.0/24, 192.168.122.0/24 (все сети где есть клиенты почтовые)
- Ограничения почтового язика в байтах: 0
- Символ расширения локальных адресов: +
- Использовать интернет протокол: 3 (ipv4)

Установка сертификата (центр сертификации этот же сервер, нужно?)

```bash
cp /certs/cacert.pem /usr/local/share/ca-certificates/cacert.crt
update-ca-certificates
```

Проверяем keytab и настиваем к нему доступ:

```bash
klist -kte /etc/dovecot/mail.keytab
```

```bash
ipa-getkeytab -p <service>/<fqdn> -k /etc/dovekot/mail.keytab
```

```bash
chmod 640 /etc/dovecot/mail.keytab
chmod dovecot:dovecot /etc/dovecot/mail.keytab
```

```bash
ls -la /etc/dovecot/mail.keytab
```

```bash
nano /etc/postfix/main.cf
```

```bash
# TLS parameters
smtpd_tls_cert_file=/cert/newcert.pem
smtpd_tls_key_file=/cert/newcert.pem
smtpd_use_tls=yes

mail_spool_directory = /var/maildir/

smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth-enevle = yes

line_legth_limit = 3072
```

```bash
mkdir -p /var/maildir/
chgrp -R mail /var/maildir/
chmor 2777 /var/maildir/
```

```bash
nano /etc/dovecot/conf.d/10-auth.conf
```

```bash
# Host name to use in GSSAPI principal names. The default is to use the
# name returned by gethostname(). Use "$ALL" (with quotes) to allow all keytab
# entries.
auth_gssapi_hostname ="$ALL"

# Kerberos keytab to use for the GSSAPI mechanism. Will use the system
# default (usually /etc/krb5.keytab) if not specified. You may need to change
# the auth service to run as root to be able to read this file.
auth_krb5_keytab = /etc/dovecot/mail.keytab

# Space separated list of wanted authentication mechanisms:
#   plain login digest-md5 cram-md5 ntlm rpa apop anonymous gssapi otp skey
#   gss-spnego
# NOTE: See also disable_plaintext_auth setting.
auth_mechanisms = gssapi
```

```bash
nano /etc/dovecot/conf.d/10-ssl.conf
```

```bash
# SSL/TLS support: yes, no, required. <doc/wiki/SSL.txt>
ssl = yes

# PEM encoded X.509 SSL/TLS certificate and private key. They're opened before
# dropping root privileges, so keep the key file unreadable by anyone but
# root. Included doc/mkcert.sh can be used to easily generate self-signed
# certificate, just make sure to update the domains in dovecot-openssl.cnf
ssl_cert = </cert/newcert.pem
ssl_key = </cert/newcert.pem
```

```bash
nano /etc/dovecot/conf.d/10-master.conf
```

```bash
service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }

  # Number of connections to handle before starting a new process. Typically
  # the only useful values are 0 (unlimited) or 1. 1 is more secure, but 0
  # is faster. <doc/wiki/LoginProcess.txt>
  #service_count = 1

  # Number of processes to always keep waiting for more connections.
  #process_min_avail = 0

  # If you set service_count=0, you probably need to grow this.
  #vsz_limit = $default_vsz_limit
}

service auth {
  # auth_socket_path points to this userdb socket by default. It's typically
  # used by dovecot-lda, doveadm, possibly imap process, etc. Users that have
  # full permissions to this socket are able to get a list of all usernames and
  # get the results of everyone's userdb lookups.
  #
  # The default 0666 mode allows anyone to connect to the socket, but the
  # userdb lookups will succeed only if the userdb returns an "uid" field that
  # matches the caller process's UID. Also if caller's uid or gid matches the
  # socket's uid or gid the lookup succeeds. Anything else causes a failure.
  #
  # To give the caller full permissions to lookup all users, set the mode to
  # something else than 0666 and Dovecot lets the kernel enforce the
  # permissions (e.g. 0777 allows everyone full permissions).
  unix_listener auth-userdb {
    # Unfortunately, getsockopt(SO_PEERCRED) doesn't work with kFreeBSD.
    # So we can work around the problem by giving everyone access to the 
    # userdb socket.
    # Cf. http://bugs.debian.org/cgi-bin/bugreport.cgi?bug=699121#10
    #mode = 0777
    #user = 
    #group = 
  }

  # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
    user = postfix
    group = postfix
  }

  # Auth process is run as this user.
  #user = $default_internal_user
}
```

```bash
nano /etc/dovecot/conf.d/15-mailboxes.conf
```


```bash
  # For \Sent mailboxes there are two widely used names. We'll mark both of
  # them as \Sent. User typically deletes one of them if duplicates are created.
  mailbox Sent {
    special_use = \Sent
    auto = subscride
  }
```


line_length_limit = 3072

И не забыть добавить mx запись

