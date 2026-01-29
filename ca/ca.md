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
cp /etc/ca/ca.crt /usr/local/share/ca-certificates/rea26-ca.crt
update-ca-certificates
```

---


## 🔐 Настройка доверия через .bashrc

```bash
# Системное доверие
cp /etc/ca/ca.crt /usr/local/share/ca-certificates/rea26-ca.crt
update-ca-certificates

# Firefox (для всех пользователей)
apt install -y libnss3-tools
for profile in /home/*/.mozilla/firefox/*.default*; do
  certutil -A -n "REA2026-CA" -t "TC,C,C" -i /etc/ca/ca.crt -d sql:$profile 2>/dev/null || true
done

# Thunderbird (для всех пользователей)
for profile in /home/*/.thunderbird/*.default*; do
  certutil -A -n "REA2026-CA" -t "TC,C,C" -i /etc/ca/ca.crt -d sql:$profile 2>/dev/null || true
done
```

## 🦊 Доверие для браузеров через /etc/skel

```bash
# 1. Создаём шаблон профиля с уже импортированным сертификатом
mkdir -p /etc/skel/.mozilla/firefox/.default-release
mkdir -p /etc/skel/.thunderbird/.default-release

# 2. Инициализируем базы сертификатов
certutil -N -d sql:/etc/skel/.mozilla/firefox/.default-release --empty-password
certutil -N -d sql:/etc/skel/.thunderbird/.default-release --empty-password

# 3. Импортируем корневой сертификат в шаблоны
certutil -A -n "REA2026-CA" -t "TC,C,C" \
  -i /etc/ca/ca.crt \
  -d sql:/etc/skel/.mozilla/firefox/.default-release

certutil -A -n "REA2026-CA" -t "TC,C,C" \
  -i /etc/ca/ca.crt \
  -d sql:/etc/skel/.thunderbird/.default-release

# 4. Создаём profiles.ini для автоматического использования шаблона
cat > /etc/skel/.mozilla/firefox/profiles.ini <<'EOF'
[General]
StartWithLastProfile=1

[Profile0]
Name=default-release
IsRelative=1
Path=.default-release
Default=1
EOF

cat > /etc/skel/.thunderbird/profiles.ini <<'EOF'
[General]
StartWithLastProfile=1

[Profile0]
Name=default-release
IsRelative=1
Path=.default-release
Default=1
EOF

# 5. Права доступа
chmod -R 700 /etc/skel/.mozilla /etc/skel/.thunderbird
```

---

## 👤 Для существующих пользователей (kda и др.)

```bash
# Импорт сертификата в профили всех текущих пользователей
for user_home in /home/*; do
  user=$(basename "$user_home")
  
  # Firefox
  profile=$(find "$user_home/.mozilla/firefox" -name "*.default*" -type d 2>/dev/null | head -1)
  [ -n "$profile" ] && certutil -A -n "REA2026-CA" -t "TC,C,C" \
    -i /etc/ca/ca.crt -d sql:"$profile" 2>/dev/null
  
  # Thunderbird
  profile=$(find "$user_home/.thunderbird" -name "*.default*" -type d 2>/dev/null | head -1)
  [ -n "$profile" ] && certutil -A -n "REA2026-CA" -t "TC,C,C" \
    -i /etc/ca/ca.crt -d sql:"$profile" 2>/dev/null
done
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
