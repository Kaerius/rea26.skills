## 📌 Этап 1: Развёртывание корпоративного ЦС (Enterprise Root CA) на Windows Server 2025

> **Важно:** В задании указано два домена: `rea26.ru` (в требовании к CRL) и `rea2026.ru` (как внешний домен компании). Для публикации CRL используйте **точно то имя, которое указано в задании — `crl.rea26.ru`**. Уточните у организаторов при первой возможности, но в рамках задания следуйте тексту: `http://crl.rea26.ru`.

---

### 🔧 Шаг 1: Установка роли Active Directory Certificate Services

1. Откройте **Server Manager** → **Manage** → **Add Roles and Features**
2. Выберите:
   - Installation Type: **Role-based or feature-based installation**
   - Server Selection: **DC** (выбрать текущий сервер)
   - Server Roles: ✅ **Active Directory Certificate Services**
     - При запросе добавить обязательные компоненты — **Add Features**
3. На экране **Role Services** выберите:
   - ✅ **Certification Authority** (обязательно)
   - ✅ **Certification Authority Web Enrollment** (для веб-интерфейса)
   - ✅ **Certificate Enrollment Web Service** (рекомендуется)
   - ✅ **Certificate Enrollment Policy Web Service** (рекомендуется)
4. Подтвердите установку → **Install**

> ⏱️ Ожидайте завершения установки (~3–5 минут)

---

### 🔐 Шаг 2: Настройка Enterprise Root CA

1. После установки в **Server Manager** появится флаг ⚠️ в правом верхнем углу → кликните → **Configure Active Directory Certificate Services**
2. Запустится **Configuration Wizard**:
   - Credentials: оставить текущую учётную запись (`reaskills2026\Administrator`)
   - Role Services: выберите **Certification Authority**
3. **Setup Type**: ✅ **Enterprise CA**
4. **CA Type**: ✅ **Root CA**
5. **Private Key**: 
   - ✅ **Create a new private key**
   - Next → использовать настройки по умолчанию (RSA, 4096-bit)
6. **Cryptographic Algorithm**:
   - Hash Algorithm: **SHA256**
   - Next
7. **CA Name**:
   - Common name: **REA2026-CA**
   - Distinguished name suffix: оставить пустым или `DC=reaskills2026,DC=local`
8. **Validity Period**: **20 years** (максимальный для корневого ЦС)
9. **Certificate Database**:
   - Database location: `C:\Windows\system32\CertLog`
   - Log location: `C:\Windows\system32\CertLog`
   - (оставить по умолчанию)
10. Нажмите **Configure** → дождитесь завершения (~2 минуты)
11. Перезагрузите сервер **только если потребует мастер** (обычно не требуется)

---

### 🌐 Шаг 3: Настройка публикации CRL

#### 3.1. Настройка путей публикации в консоли CA

1. Откройте **Certification Authority** (`certsrv.msc`)
2. ПКМ по **REA2026-CA** → **Properties**
3. Вкладка **Extensions**:
   - Выделите **CDP** → нажмите **Add...**
   - Введите:
     ```
     http://crl.rea26.ru/crl/%3%8.crl
     ```
   - ✅ **Include in the CDP extension of issued certificates**
   - ✅ **Include in all CRLs. Clients use this to find Delta CRLs**
   - ✅ **Publish CRLs to this location**
   - ✅ **Publish Delta CRLs to this location**
   - OK
4. Повторите для **AIA** (Authority Information Access):
   - Добавьте:
     ```
     http://crl.rea26.ru/cert/%3%8.crt
     ```
   - ✅ **Include in the AIA extension of issued certificates**
   - ✅ **Publish certificates to this location**
5. Вкладка **General** → **Copy** → **Copy all templates** → OK

#### 3.2. Создание общей папки для публикации

```powershell
# Создаём папки
New-Item -Path "C:\CertEnroll" -ItemType Directory -Force
New-Item -Path "C:\CertEnroll\crl" -ItemType Directory -Force
New-Item -Path "C:\CertEnroll\cert" -ItemType Directory -Force

# Настраиваем общие ресурсы
New-SmbShare -Name "CertEnroll" -Path "C:\CertEnroll" -FullAccess "Everyone"
New-SmbShare -Name "crl" -Path "C:\CertEnroll\crl" -ReadAccess "Everyone"
New-SmbShare -Name "cert" -Path "C:\CertEnroll\cert" -ReadAccess "Everyone"

# Настраиваем права NTFS
$acl = Get-Acl "C:\CertEnroll"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule("Everyone","FullControl","ContainerInherit,ObjectInherit","None","Allow")
$acl.SetAccessRule($rule)
Set-Acl -Path "C:\CertEnroll" -AclObject $acl
```

#### 3.3. Настройка IIS для веб-публикации CRL

1. Установите роль **Web Server (IIS)** если не установлена:
   ```powershell
   Install-WindowsFeature -Name Web-Server -IncludeManagementTools
   ```
2. Откройте **IIS Manager** (`inetmgr`)
3. В левом дереве: **Sites** → **Default Web Site**
4. Удалите стандартные привязки (порт 80/443 к `*`)
5. Добавьте новую привязку:
   - Type: **http**
   - IP address: **192.168.100.1** (внутренний адрес DC)
   - Port: **80**
   - Host name: **crl.rea26.ru**
6. Создайте виртуальные директории:
   - ПКМ на **Default Web Site** → **Add Virtual Directory...**
     - Alias: `crl`
     - Physical path: `C:\CertEnroll\crl`
   - Повторите для `cert` → `C:\CertEnroll\cert`
7. Перезапустите сайт: ПКМ → **Manage Website** → **Restart**

---

### 🔄 Шаг 4: Настройка маршрутизации для внешнего доступа к CRL

> CRL должен быть доступен из внешней сети по `http://crl.rea26.ru`. Так как DC находится во внутренней сети (CORP), нужен проброс через шлюз.

#### На сервере **RRAS1**:

1. Настройте статическую запись DNS для внешней сети:
   ```powershell
   # На RRAS1 (внешний интерфейс)
   Add-DnsServerResourceRecordA -Name "crl" -ZoneName "rea26.ru" -IPv4Address "192.168.122.x" -ComputerName "RRAS1"
   ```
   *(замените `192.168.122.x` на реальный внешний адрес RRAS1)*

2. Настройте проброс порта 80 с внешнего интерфейса на DC:
   ```powershell
   # На RRAS1 (PowerShell от имени администратора)
   netsh interface portproxy add v4tov4 listenport=80 listenaddress=0.0.0.0 connectport=80 connectaddress=192.168.100.1
   ```

3. Разрешите трафик в брандмауэре:
   ```powershell
   New-NetFirewallRule -DisplayName "Allow HTTP to DC" -Direction Inbound -Protocol TCP -LocalPort 80 -Action Allow
   ```

---

### ✅ Шаг 5: Принудительная публикация первого CRL и тестирование

1. Принудительно опубликуйте CRL:
   ```powershell
   certutil -crl
   ```
2. Проверьте появление файлов:
   ```powershell
   dir C:\CertEnroll\crl\
   # Должен появиться файл вида: REA2026-CA.crl
   ```
3. Проверьте локальный доступ:
   ```powershell
   Invoke-WebRequest -Uri "http://crl.rea26.ru/crl/REA2026-CA.crl" -OutFile "$env:TEMP\test.crl"
   certutil -dump "$env:TEMP\test.crl"
   ```
4. Проверьте из внешней сети (с `CLI-EXT`):
   ```powershell
   # На CLI-EXT
   curl http://crl.rea26.ru/crl/REA2026-CA.crl -o test.crl
   certutil -dump test.crl
   ```
   → Должен отобразиться корректный CRL без ошибок 404/403

---

### ⚠️ Критические проверки перед переходом к следующему этапу

| Проверка | Команда/Действие | Ожидаемый результат |
|----------|------------------|---------------------|
| Статус службы CA | `Get-Service certsvc` | Status = **Running** |
| Наличие корневого сертификата | `certutil -store My` | Виден сертификат **REA2026-CA** |
| Доступность CDP в сертификате | `certutil -viewstore -user My` → выбрать сертификат → **View Certificate** → **Details** → **CRL Distribution Points** | Содержит `http://crl.rea26.ru/crl/...` |
| Доступность CRL извне | С `CLI-EXT`: открыть в браузере `http://crl.rea26.ru/crl/` | Отображается список файлов или скачивается `.crl` |

---

### 💡 Советы по устранению типичных проблем

| Проблема | Решение |
|----------|---------|
| `Access is denied` при публикации CRL | Проверьте права на `C:\CertEnroll` — должна быть полная контроль для `NETWORK SERVICE` |
| CRL не публикуется в указанную папку | Выполните `certutil -setreg CA\CRLPublicationURLs "1:http://crl.rea26.ru/crl/%3%8.crl\n2:file://\\dc\CertEnroll\%3%8.crl"` + перезапустите службу `certsvc` |
| 404 ошибка в браузере | Проверьте привязки IIS — должен быть хостнейм `crl.rea26.ru` и правильный путь к виртуальной директории |
| Недоступность из внешней сети | Проверьте:<br>1) Запись DNS на RRAS1<br>2) Правила проброса портов (`netsh interface portproxy show all`)<br>3) Брандмауэр на RRAS1 и DC |

> ✅ После успешного прохождения всех проверок переходите к **Этапу 2 (запрос сертификатов для ADFS/WAP/RDS)**. Без работающего ЦС и доступного CRL все последующие этапы завершатся ошибками сертификатов.
