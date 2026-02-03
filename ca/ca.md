## 🔑 Универсальный сертификат (один на все сервисы)
# 1. Установка и подготовка

```bash
apt install easy-rsa
```

# 2. Инициализация ЦС

```bash
cd /usr/share/easy-rsa/
./easyrsa --pki-dir=/etc/ca init-pki
```

# 3. Корневой сертификат (CN=REA2026-CA)

```bash
./easyrsa --pki-dir=/etc/ca build-ca nopass
```
> Common Name: REA2026-CA

# 4. УНИВЕРСАЛЬНЫЙ сертификат для всех сервисов

```bash
./easyrsa --pki-dir=/etc/ca --subject-alt-name="DNS:*.rea26.skills,DNS:*.rea26.ru" build-server-full rea26 nopass
```

---

## 🔐 Доверие на уровне системы (все устройства)

```bash
cp /etc/ca/ca.crt /usr/local/share/ca-certificates/
dpkg-reconfigure ca-certificates
```

---

## 📁 Итоговая структура

```
/etc/ca/
├── ca.crt          # ← Корневой сертификат (раздавать всем!)
├── ca.key          # ← Закрытый ключ ЦС (не передавать!)
├── issued/rea26.crt    # ← Универсальный сертификат для всех сервисов
└── private/rea26.key   # ← Закрытый ключ универсального сертификата

/etc/skel/
├── .mozilla/firefox/.default-release/
│   ├── cert9.db        # ← База с импортированным REA2026-CA
│   └── profiles.ini
└── .thunderbird/.default-release/
    ├── cert9.db        # ← База с импортированным REA2026-CA
    └── profiles.ini
```

---

## ✅ Проверка

```bash
# 1. Проверка SAN в сертификате
openssl x509 -in /etc/ca/issued/rea26.crt -text -noout | grep -A2 "X509v3 Subject Alternative Name"

# 2. Проверка доверия в системе
curl -I https://cr-srv.rea26.skills  # без --insecure!

# 3. Проверка доверия в профиле
certutil -L -d sql:/etc/skel/.mozilla/firefox/.default-release | grep REA2026-CA
```
