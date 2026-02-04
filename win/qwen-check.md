## 📋 ПАМЯТКА: Выполнение задания ReaSkills 2026 — Модуль В (Windows)
*Только действия «куда нажать» — без пояснений*

---

### 🔹 ЭТАП 0: Проверка исходного состояния
```powershell
ipconfig /all
nslookup reaskills2026.local
ping dc.reaskills2026.local
```

---

### 🔹 ЭТАП 1: ЦС на DC (192.168.100.1)

#### 1.1 Установка роли
```
Server Manager → Manage → Add Roles and Features
→ Role-based installation → DC
→ ✅ Active Directory Certificate Services → Add Features → Next
→ Role Services:
   ✅ Certification Authority
   ✅ Certification Authority Web Enrollment
   ✅ Certificate Enrollment Web Service
   ✅ Certificate Enrollment Policy Web Service
→ Next → Next → Install
```

#### 1.2 Настройка ЦС
```
Server Manager → ⚠️ (Notifications) → Post-deployment Configuration → Run
→ Configure Active Directory Certificate Services
→ Credentials: оставить текущие → Next
→ Role Services: ✅ Certification Authority → Next
→ Setup Type: Enterprise CA → Next
→ CA Type: Root CA → Next
→ Private Key: Create a new private key → Next
→ Cryptography: RSA 4096-bit, SHA256 → Next
→ CA Name: REA2026-CA → Next
→ Validity Period: 20 years → Next
→ Database: оставить по умолчанию → Next → Configure
```

#### 1.3 Настройка публикации CRL (локально)
```powershell
New-Item -Path "C:\CertEnroll" -ItemType Directory -Force
certutil -setreg CA\CRLPublicationURLs "1:file://\\?\C:\CertEnroll\%3%8.crl"
certutil -setreg CA\CACertPublicationURLs "1:file://\\?\C:\CertEnroll\%3%8.crt"
Restart-Service certsvc -Force
certutil -crl
```

#### 1.4 На RRAS1 — подготовка веб-сервера для CRL
```powershell
# Установка IIS:
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

# Создание папки и общей папки:
New-Item -Path "C:\inetpub\wwwroot\crl" -ItemType Directory -Force
New-SmbShare -Name "crlpub" -Path "C:\inetpub\wwwroot\crl" -ReadAccess "Everyone"
icacls "C:\inetpub\wwwroot\crl" /grant Everyone:(F)
```

#### 1.5 Синхронизация CRL (на DC)
```powershell
$action = New-ScheduledTaskAction -Execute "robocopy.exe" -Argument '"C:\CertEnroll" "\\rras1.reaskills2026.local\crlpub" *.crl *.crt /MIR /R:2 /W:5'
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date) -RepetitionInterval (New-TimeSpan -Minutes 5) -RepetitionDuration ([TimeSpan]::MaxValue)
Register-ScheduledTask -TaskName "CRL_Sync" -Action $action -Trigger $trigger -User "SYSTEM" -RunLevel Highest
```

#### 1.6 Добавление внешнего URL в свойствах ЦС
```
certsrv.msc → ПКМ на REA2026-CA → Properties
→ Extensions → CDP → Add...
   http://crl.rea2026.ru/crl/%3%8.crl
   ✅ Include in the CDP extension of issued certificates
   ❌ Publish CRLs to this location
→ Apply

→ Extensions → AIA → Add...
   http://crl.rea2026.ru/cert/%3%8.crt
   ✅ Include in the AIA extension of issued certificates
   ❌ Publish certificates to this location
→ Apply → OK

Restart-Service certsvc -Force
certutil -crl
```

#### 1.7 Проверка
```powershell
# На DC:
dir C:\CertEnroll\*.crl

# На RRAS1 (через 5 мин):
dir C:\inetpub\wwwroot\crl\*.crl

# На CLI-EXT:
nslookup crl.rea2026.ru
curl http://crl.rea2026.ru/crl/
```

---

### 🔹 ЭТАП 2: Сертификаты

#### 2.1 Запрос сертификата для ADFS (adfs.rea2026.ru)
```
Браузер на ADFS → https://dc/certsrv
→ Request a certificate → Advanced certificate request
→ Create and submit a request → Fill in manually
   Name: adfs.rea26.ru
   Type: Web Server
   Key size: 4096
   ✅ Make private key exportable
→ Submit → Download certificate → Сохранить как adfs.cer
→ Download certificate chain → Сохранить как adfs.p7b
```

#### 2.2 Запрос сертификата для WAP (rdweb.rea2026.ru)
```
Браузер на WAP → https://dc/certsrv
→ Request a certificate → Advanced certificate request
→ Create and submit a request → Fill in manually
   Name: rdweb.rea2026.ru
   Type: Web Server
   Key size: 4096
   ✅ Make private key exportable
→ Submit → Download certificate → Сохранить как rdweb.cer
→ Download certificate chain → Сохранить как rdweb.p7b
```

#### 2.3 Запрос сертификата для RDS (rds.reaskills2026.local)
```
Браузер на RDS → https://dc/certsrv
→ Request a certificate → Advanced certificate request
→ Create and submit a request → Fill in manually
   Name: rds.reaskills2026.local
   Type: Web Server
   Key size: 4096
   ✅ Make private key exportable
→ Subject name → Type: Common name → Value: rds.reaskills2026.local
→ Extensions → DNS name → Add → rds.reaskills2026.local → Add
→ Submit → Download certificate → Сохранить как rds.cer
→ Download certificate chain → Сохранить как rds.p7b
```

#### 2.4 Установка сертификатов
```
На каждом сервере:
   ПКМ на .cer → Install Certificate → Local Machine → Place all certificates in the following store → Personal → Finish
   ПКМ на .p7b → Install Certificate → Local Machine → Place all certificates in the following store → Trusted Root Certification Authorities → Finish
```

---

### 🔹 ЭТАП 3: ADFS на ADFS (192.168.100.2)

#### 3.1 Установка роли
```
Server Manager → Add Roles and Features
→ ✅ Active Directory Federation Services → Next → Next → Install
```

#### 3.2 Настройка фермы
```
Server Manager → ⚠️ → Post-deployment Configuration → Run the AD FS Management snap-in
→ Action → Configure Federation Service on this server
→ Create the first federation server in a federation server farm → Next
→ SSL Certificate: выбрать adfs.rea26.ru → Next
→ Federation Service Display Name: REA ADFS → Next
→ Service account: Use the built-in account (adfssrv) → Next
→ Primary Federation server name: adfs.rea26.ru → Next → Next → Configure
```

#### 3.3 Включение проверки CRL и регистрации
```powershell
Set-AdfsProperties -EnableCRLChecking $true
Set-AdfsWebConfig -PageLayoutVersion 2
Set-AdfsProperties -EnableLocalAuthenticationTypes $true
Restart-Service adfssrv
```

#### 3.4 Проверка
```
Браузер на CLI-INT: https://adfs.rea26.ru/adfs/ls/idpinitiatedsignon.htm
```

---

### 🔹 ЭТАП 4: WAP на WAP (192.168.50.1)

#### 4.1 Установка роли
```
Server Manager → Add Roles and Features
→ ✅ Remote Access → Next
→ Role Services: ✅ Web Application Proxy → Next → Next → Install
```

#### 4.2 Присоединение к ферме ADFS
```
Server Manager → ⚠️ → Open the Web Application Proxy Configuration Wizard
→ Federation Service: https://adfs.rea26.ru
→ Учётные данные: reaskills2026\Administrator / P@ssw0rd → Next
→ SSL Certificate: выбрать rdweb.rea26.ru → Next → Configure
```

#### 4.3 Публикация RDS
```
Web Application Proxy → Publish → Publish new application through AD FS
→ Relying party name: RDS
→ External URL: https://rdweb.rea26.ru
→ Internal URL: https://rds.reaskills2026.local
→ Pre-authentication: AD FS
→ Relying party identifier: https://rds.reaskills2026.local
→ Next → Publish
```

#### 4.4 Включение аутентификации по сертификату
```
IIS Manager → Sites → Default Web Site → SSL Settings
→ ✅ Require SSL → Client certificates: Accept → Apply
```

---

### 🔹 ЭТАП 5: RDS на RDS (192.168.200.1)

#### 5.1 Установка ролей
```
Server Manager → Add Roles and Features
→ ✅ Remote Desktop Services → Next
→ Deployment Type: Standard deployment → Next
→ Role Services:
   ✅ RD Connection Broker
   ✅ RD Web Access
   ✅ RD Session Host
→ Next → Next → Install
```

#### 5.2 Установка мессенджера MAX
```
Скачать установщик MAX → запустить от имени администратора → установить по умолчанию
```

#### 5.3 Настройка сертификата RDS
```
Server Manager → Remote Desktop Services → Deployment Overview
→ Tasks → Edit Deployment Properties → Certificates
→ RD Web Access → Select existing certificate → указать rds.cer + пароль
→ ✅ Allow the certificate to be added to Trusted Root Certification Authorities certificate store → Apply

→ RD Connection Broker → Select existing certificate → указать rds.cer + пароль → Apply

→ RD Session Host → Select existing certificate → указать rds.cer + пароль → Apply
```

#### 5.4 HTTPS-only с редиректом HTTP→HTTPS
```powershell
Install-WindowsFeature Web-Url-Rewrite2
```
```
IIS Manager → Sites → Default Web Site → URL Rewrite → Add Rule(s)...
→ Blank rule → Name: HTTP to HTTPS redirect
→ Match URL → Requested URL: Matches the Pattern → Using: Wildcards → Pattern: *
→ Conditions → Add → {HTTPS} → Matches the Pattern → off
→ Action → Action type: Redirect → Redirect URL: https://{HTTP_HOST}{REQUEST_URI}
→ Redirect type: Permanent (301) → Apply
```

#### 5.5 Настройка делегирования Kerberos (на DC)
```
Active Directory Users and Computers → View → ✅ Advanced Features
→ Computers → ПКМ на RDS → Properties → Delegation
→ ✅ Trust this computer for delegation to specified services only
→ ✅ Use any authentication protocol
→ Add → User or Computer → RDS → OK
→ Services: ✅ HTTP → OK → OK
```

#### 5.6 Публикация приложения MAX
```
Server Manager → Remote Desktop Services → Collections
→ Tasks → Create Session Collection
→ Name: MAX → Next
→ Host servers: RDS → Next → Next → Finish

→ Collections → MAX → RemoteApp Programs → Tasks → Publish RemoteApp Programs
→ Выбрать "MAX" → Publish
```

#### 5.7 Ярлык .rdp на рабочий стол (через GPO на DC)
```powershell
# Создать файл C:\MAX.rdp с содержимым:
full address:s:rds.reaskills2026.local
prompt for credentials:i:0
authentication level:i:2
remoteapplicationmode:i:1
remoteapplicationprogram:s:||MAX
remoteapplicationname:s:MAX
use multimon:i:0
audiomode:i:0
disable wallpaper:i:1
disable full window drag:i:1
disable menu anims:i:1
disable themes:i:0
alternate shell:s:
shell working directory:s:
```
```
gpmc.msc → Group Policy Objects → New → Name: RDP_Shortcut → OK
→ Edit → User Configuration → Preferences → Windows Settings → Files
→ ПКМ → New → File
→ Action: Create
→ Source file(s): C:\MAX.rdp
→ Destination Folder: %Public%\Desktop
→ Apply → OK

→ Scope → Security Filtering → Remove Authenticated Users → Add → Domain Users → OK
→ Link to domain reaskills2026.local
```

#### 5.8 Добавление отпечатка сертификата в доверенные (на DC)
```powershell
certutil -store My "rds.reaskills2026.local"
# Скопировать отпечаток (без пробелов!)
```
```
gpmc.msc → RDP_Shortcut → Edit
→ Computer Configuration → Policies → Administrative Templates → Windows Components → Remote Desktop Services → Remote Desktop Connection Client
→ Specify SHA1 thumbprints of certificates representing trusted .rdp publishers → Enabled
→ Value: [вставить отпечаток без пробелов] → OK
```

#### 5.9 Добавление RDS в интранет-зону (на DC)
```
gpmc.msc → RDP_Shortcut → Edit
→ User Configuration → Policies → Administrative Templates → Windows Components → Internet Explorer → Security Page
→ Site to Zone Assignment List → Enabled → Show...
   Value name: https://rds.reaskills2026.local → Value: 1
   Value name: rds.reaskills2026.local → Value: 1
→ OK
```

---

### 🔹 ЭТАП 6: Групповые политики для Edge (на DC)

#### 6.1 Установка ADMX-шаблонов
```
Скачать: https://www.microsoft.com/en-us/edge/business/download → CAB
→ Распаковать CAB → распаковать ZIP внутри
→ Скопировать:
   windows\admx\msedge.admx → C:\Windows\PolicyDefinitions\
   windows\admx\en-US\msedge.adml → C:\Windows\PolicyDefinitions\en-US\
```

#### 6.2 Создание GPO
```
gpmc.msc → Group Policy Objects → New → Name: Edge_Policy → OK
→ Edit

User Configuration → Policies → Administrative Templates → Microsoft Edge
→ Startup, home page and new tab page → Configure the Start pages → Enabled
   Primary start page: https://rds.reaskills2026.local → OK

→ Startup, home page and new tab page → Hide First Run experience → Enabled → OK

→ Configure the new tab page → Hide the First Run experience → Enabled → OK

→ Configure the new tab page → Show recommendations in Start and New Tab pages → Disabled → OK

→ Settings → Hide default browser prompt → Enabled → OK
```

#### 6.3 Применение к клиентам
```
GPO → Scope → Security Filtering → Remove Authenticated Users → Add → Domain Computers → OK
→ Link to domain reaskills2026.local
```

---

### 🔹 ЭТАП 7: Аутентификация по сертификату (наружу)

#### 7.1 Создание шаблона пользовательского сертификата (на DC)
```
certsrv.msc → Certificate Templates → Manage
→ ПКМ на "User" → Duplicate Template
→ Compatibility: Windows Server 2025
→ General → Template display name: UserCert → Validity period: 2 years
→ Request Handling → ✅ Allow private key to be exported
→ Extensions → Application Policies → Edit → ✅ Client Authentication → OK
→ Security → Add → Domain Users → ✅ Enroll → OK → OK

certsrv.msc → Certificate Templates → ПКМ → New → Certificate Template to Issue → UserCert → OK
```

#### 7.2 Выпуск сертификата для пользователя (на CLI-EXT)
```
certmgr.msc → Personal → Certificates → All Tasks → Request New Certificate
→ Next → Active Directory Enrollment Policy → Next
→ Выбрать "UserCert" → Enroll → Finish

→ Personal → Certificates → ПКМ на сертификат → All Tasks → Export
→ ✅ Yes, export the private key → Next → Next → Set password → Next → Finish
→ Сохранить как user.pfx
```

#### 7.3 Установка на CLI-EXT
```
ПКМ на user.pfx → Install PFX → Current User → Next → Enter password → Next → Next → Finish
```

---

### 🔹 ЭТАП 8: Финальная проверка

| Проверка | Где | Действие |
|----------|-----|----------|
| Edge стартовая страница | CLI-INT | `gpupdate /force` → запустить Edge |
| RDP-ярлык изнутри | CLI-INT | Двойной клик по ярлыку MAX на рабочем столе |
| SSO RDS изнутри | CLI-INT | Браузер → `https://rds.reaskills2026.local` |
| CRL извне | CLI-EXT | `nslookup crl.rea2026.ru` → `curl http://crl.rea2026.ru/crl/` |
| Внешний доступ | CLI-EXT | Браузер → `https://rdweb.rea26.ru` → вход → скачать .rdp |
| RDP извне | CLI-EXT | Запустить скачанный .rdp файл |
| Аутентификация по сертификату | CLI-EXT | В браузере выбрать сертификат при входе в ADFS |

---

### ✅ ФИНАЛЬНЫЙ ЧЕКЛИСТ

```
[ ] Имя ЦС: REA2026-CA
[ ] CRL: http://crl.rea2026.ru/crl/ доступен из CLI-EXT
[ ] ADFS имя портала: REA ADFS (в заголовке страницы)
[ ] ADFS: проверка CRL включена (Set-AdfsProperties -EnableCRLChecking $true)
[ ] Внешние имена: adfs.rea2026.ru, rdweb.rea2026.ru (НЕ .local!)
[ ] RDS: HTTP → HTTPS редирект работает
[ ] SSO изнутри: вход без пароля на https://rds.reaskills2026.local
[ ] RDP изнутри: запуск без пароля и предупреждений
[ ] RDP извне: запуск с запросом пароля
[ ] Edge: стартовая страница + нет приветствия + нет рекомендаций
[ ] Сертификаты: все от REA2026-CA
[ ] Нет проброса портов на DC (netsh interface portproxy show all → пусто)
```

> ⚠️ **ЗАПРЕЩЕНО:** Использовать `.local` во внешней сети. Прямой доступ к внутренним серверам извне.
