## Актуализированная инструкция по работе с EasyRSA для ReaSkills 2026

### 📋 Полный список сертификатов для генерации

| Сервис | Доменные имена (SAN) | Где развернуть | Файл сертификата |
|--------|----------------------|----------------|------------------|
| **Почтовый сервер** | `cr-srv.rea26.skills` | CR-SRV | `/etc/ca/issued/mail.crt` |
| **ISP веб-сайт** | `isp.rea26.skills`, `isp.rea26.ru` | ISP-SRV | `/etc/ca/issued/isp.crt` |
| **Registry k8s** | `registry.rea26.skills` | BR-SRV* (ingress) | `/etc/ca/issued/registry.crt` |
| **Корпоративный портал** | `portal.rea26.skills`, `portal.rea26.ru` | BR-SRV* (ingress) | `/etc/ca/issued/portal.crt` |
| **Grafana** | `grafana.rea26.skills` | BR-SRV* (ingress) | `/etc/ca/issued/grafana.crt` |
| **Корневой сертификат** | `REA2026-CA` | Все устройства | `/etc/ca/ca.crt` |

> 💡 **Важно**: Для сервисов с двумя доменами (ISP, Portal) **обязательно** использовать один сертификат с несколькими SAN, иначе браузеры будут ругаться на несоответствие имени.

---

### 🔧 Пошаговая инструкция (выполнять на CR-SRV)

#### 1. Установка и подготовка
```bash
# Настройка прокси (обязательно для загрузки пакетов)
export http_proxy="http://proxy.tech.skills:3128"
export https_proxy="http://proxy.tech.skills:3128"

# Установка пакета
apt update
apt install easy-rsa -y

# Переход в рабочую директорию
cd /usr/share/easy-rsa/
```

#### 2. Инициализация центра сертификации
```bash
# Создание PKI в требуемом каталоге
./easyrsa --pki-dir=/etc/ca init-pki

# Генерация корневого сертификата БЕЗ пароля (для автоматизации)
./easyrsa --pki-dir=/etc/ca build-ca nopass
```
При запросе `Common Name` введите: **`REA2026-CA`**

#### 3. Генерация сертификатов для сервисов

```bash
# 1. Почтовый сервер (без SAN - одно имя)
./easyrsa --pki-dir=/etc/ca build-server-full cr-srv.rea26.skills nopass

# 2. ISP сайт (два домена в одном сертификате!)
./easyrsa --pki-dir=/etc/ca \
  --subject-alt-name="DNS:isp.rea26.skills,DNS:isp.rea26.ru" \
  build-server-full isp nopass

# 3. Registry k8s
./easyrsa --pki-dir=/etc/ca build-server-full registry.rea26.skills nopass

# 4. Корпоративный портал (два домена)
./easyrsa --pki-dir=/etc/ca \
  --subject-alt-name="DNS:portal.rea26.skills,DNS:portal.rea26.ru" \
  build-server-full portal nopass

# 5. Grafana
./easyrsa --pki-dir=/etc/ca build-server-full grafana.rea26.skills nopass
```

> ⚠️ **Важно при генерации**:  
> - Для всех команд отвечайте `yes` при запросе подтверждения  
> - Закрытые ключи будут в `/etc/ca/private/`  
> - Сертификаты — в `/etc/ca/issued/`  
> - Не удаляйте файлы `ca.crt` и `ca.key` — они нужны для выпуска новых сертификатов!

---

### 🔐 Настройка доверия на всех устройствах

#### На серверах (CR-SRV, BR-SRV*, ISP-SRV, CR-DC, BR-DC):
```bash
# Копирование корневого сертификата
cp /etc/ca/ca.crt /usr/local/share/ca-certificates/rea2026-ca.crt

# Обновление системного хранилища
update-ca-certificates

# Проверка (должен появиться в списке)
ls -la /etc/ssl/certs/ | grep rea26
```

#### На клиентских ПК (CR-CLI, BR-CLI, OUT-CLI):
```bash
# Системное доверие (как на серверах)
cp /etc/ca/ca.crt /usr/local/share/ca-certificates/rea26-ca.crt
update-ca-certificates

# Доверие для Firefox (для ВСЕХ пользователей)
apt install libnss3-tools -y
for profile in /home/*/.mozilla/firefox/*.default*; do
  certutil -A -n "REA2026-CA" -t "TC,C,C" -i /etc/ca/ca.crt -d sql:$profile
done

# Доверие для Thunderbird (аналогично)
for profile in /home/*/.thunderbird/*.default*; do
  certutil -A -n "REA2026-CA" -t "TC,C,C" -i /etc/ca/ca.crt -d sql:$profile
done
```

> 💡 **Упрощение для учебного стенда**:  
> Можно скопировать сертификат в профиль пользователя `kda` (или другого тестового пользователя), а не для всех:
> ```bash
> certutil -A -n "REA2026-CA" -t "TC,C,C" -i /etc/ca/ca.crt -d sql:/home/kda/.mozilla/firefox/*.default-release
> ```

---

### 📁 Структура каталога `/etc/ca` после генерации
```
/etc/ca/
├── ca.crt          # ← КОРНЕВОЙ СЕРТИФИКАТ (раздавать всем!)
├── ca.key          # ← Закрытый ключ CA (хранить в секрете!)
├── issued/
│   ├── cr-srv.rea26.skills.crt
│   ├── isp.crt
│   ├── registry.rea26.skills.crt
│   ├── portal.crt
│   └── grafana.rea26.skills.crt
├── private/
│   ├── ca.key
│   ├── cr-srv.rea26.skills.key
│   ├── isp.key
│   └── ... (остальные закрытые ключи)
└── ...
```

---

### ⚙️ Как использовать сертификаты в сервисах

#### Для веб-серверов (nginx/apache на ISP-SRV):
```nginx
ssl_certificate /etc/ca/issued/isp.crt;
ssl_certificate_key /etc/ca/private/isp.key;
ssl_client_certificate /etc/ca/ca.crt;  # для клиентской аутентификации (если нужна)
```

#### Для почтового сервера (Postfix):
```bash
smtpd_tls_cert_file = /etc/ca/issued/cr-srv.rea26.skills.crt
smtpd_tls_key_file = /etc/ca/private/cr-srv.rea26.skills.key
smtpd_tls_CAfile = /etc/ca/ca.crt
```

#### Для ingress в k8s (пример манифеста):
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: portal
spec:
  tls:
  - hosts:
    - portal.rea26.skills
    - portal.rea26.ru
    secretName: portal-tls
  rules:
  - host: portal.rea26.skills
    http: ...
```
Создать secret:
```bash
kubectl create secret tls portal-tls \
  --cert=/etc/ca/issued/portal.crt \
  --key=/etc/ca/private/portal.key \
  -n default
```

---

### ✅ Проверка работоспособности
```bash
# Проверка цепочки доверия
openssl verify -CAfile /etc/ca/ca.crt /etc/ca/issued/isp.crt

# Проверка SAN в сертификате
openssl x509 -in /etc/ca/issued/isp.crt -text -noout | grep -A1 "Subject Alternative Name"

# Проверка через curl (должен работать БЕЗ --insecure)
curl -v https://isp.rea26.skills
```

> 💡 **Совет для стенда**: Если времени мало — сгенерируйте сначала сертификаты для **ISP-SRV** и **портала**, так как они критичны для проверки внешнего/внутреннего доступа. Остальные можно доделать позже.
