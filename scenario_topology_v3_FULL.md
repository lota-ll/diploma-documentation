# EcoCharge CTF Scenario - Network Topology Documentation

## Документ: Опис мережевої інфраструктури Cyber Range

**Версія:** 3.0 (Виправлено та покращено)  
**Дата:** 9 лютого 2026  
**Статус:** ✅ Актуально для розгортання

---

## 1. Концепція Архітектури

Інфраструктура EcoCharge реалізує **5-рівневу модель сегментації мережі** згідно з принципами Defense in Depth. Архітектура імітує реальну інфраструктуру регіонального оператора мережі зарядних станцій для електротранспорту.

**Ключова особливість:** Веб-сервер винесено в окрему зону "Frontend" для ізоляції публічних сервісів від управлінської DMZ, що відповідає сучасним практикам кібербезпеки.

### Мережева сегментація:

1. **Internet / WAN (192.168.125.0/24)** — Мережа атакуючого
2. **Frontend Zone (192.168.250.0/24)** — Публічний веб-додаток EcoCharge
3. **Management DMZ (192.168.100.0/24)** — Сервіси управління та моніторингу
4. **Internal Zone (192.168.20.0/24)** — База даних та ядро CSMS
5. **OT Network (172.16.0.0/24)** — Operational Technology (зарядні станції)

---

## 2. Мережева Топологія

### 2.1 Візуальна діаграма

```mermaid
graph TB
    %% Стилізація
    classDef attacker fill:#ffcccc,stroke:#cc0000,stroke-width:3px,color:#000
    classDef frontend fill:#e3f2fd,stroke:#2196f3,stroke-width:2px,color:#000
    classDef dmz fill:#fff9c4,stroke:#fbc02d,stroke-width:2px,color:#000
    classDef internal fill:#e8f5e9,stroke:#4caf50,stroke-width:2px,color:#000
    classDef ot fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px,color:#000

    subgraph Zone0["🌐 ZONE 0: INTERNET"]
        Attacker["👤 Attacker<br/>Kali Linux<br/>192.168.125.100"]:::attacker
    end

    subgraph Zone1["🌍 ZONE 1: FRONTEND"]
        WebServer["🖥️ Web Server<br/>192.168.250.50<br/>Next.js 15 + React 19<br/>📍 FLAGS: #1, #2, #3"]:::frontend
    end

    subgraph Zone2["🛡️ ZONE 2: MANAGEMENT DMZ"]
        APIGateway["⚙️ API Gateway<br/>192.168.100.20:3000<br/>Node.js/Express<br/>📍 FLAG: #4"]:::dmz
        Grafana["📊 Grafana<br/>192.168.100.30:3000<br/>Monitoring Dashboard<br/>📍 FLAG: #5"]:::dmz
        JumpHost["🔑 Jump Host<br/>192.168.100.40 (DMZ)<br/>192.168.20.40 (Internal)<br/>172.16.0.10 (OT)<br/>📍 FLAG: #6"]:::dmz
    end

    subgraph Zone3["🗄️ ZONE 3: INTERNAL"]
        CSMS["💾 CitrineOS CSMS<br/>192.168.20.20<br/>PostgreSQL + Hasura<br/>📍 FLAG: #7"]:::internal
    end

    subgraph Zone4["⚡ ZONE 4: OT NETWORK"]
        CP001["🔌 EV Charger CP001<br/>172.16.0.40<br/>OCPP 1.6-J<br/>📍 FLAG: #8"]:::ot
        CP002["🔌 EV Charger CP002<br/>172.16.0.60<br/>OCPP 2.0.1<br/>📍 FLAG: #9"]:::ot
    end

    %% Зв'язки
    Attacker ==>|"HTTP/HTTPS<br/>RCE Exploit"| WebServer
    WebServer -->|"API Requests<br/>:3000"| APIGateway
    WebServer -.->|"SSH Pivot<br/>:22 (w/ key)"| JumpHost
    WebServer -.-x|"❌ BLOCKED"| Grafana
    WebServer -.-x|"❌ BLOCKED"| CSMS
    
    APIGateway -->|"GraphQL/REST"| CSMS
    Grafana -->|"PostgreSQL<br/>:5432"| CSMS
    JumpHost -->|"Admin Access"| CSMS
    JumpHost -->|"SSH Tunnel"| Grafana
    JumpHost -->|"OCPP Sniffing<br/>Manipulation"| CP001
    JumpHost -->|"OCPP Commands"| CP002
    
    CP001 -->|"OCPP 1.6<br/>ws://192.168.20.20:8080"| CSMS
    CP002 -->|"OCPP 2.0.1<br/>ws://192.168.20.20:8092"| CSMS
```

### 2.2 Таблиця мережевих сегментів

| Зона            | CIDR             | VLAN | Gateway         | Призначення                    | Правила доступу                                                    |
|-----------------|------------------|------|-----------------|--------------------------------|--------------------------------------------------------------------|
| **Internet**    | 192.168.125.0/24 | WAN  | ISP Router      | Мережа атакуючого              | ✅ Access to Frontend only<br/>❌ All other zones blocked          |
| **Frontend**    | 192.168.250.0/24 | 50   | 192.168.250.1   | Публічний веб-сайт EcoCharge   | ✅ To DMZ (ports 3000, 22)<br/>❌ To Internal/OT (DROP)            |
| **DMZ**         | 192.168.100.0/24 | 20   | 192.168.100.1   | Управління та моніторинг       | ✅ From Frontend<br/>✅ To Internal (specific ports)<br/>✅ To OT (via Jump Host) |
| **Internal**    | 192.168.20.0/24  | 30   | 192.168.20.1    | Backend, Database, CSMS        | ✅ From DMZ only<br/>✅ To OT (OCPP WebSocket)                     |
| **OT**          | 172.16.0.0/24    | 40   | 172.16.0.1      | Зарядні станції (Operational Technology) | ✅ To CSMS only<br/>✅ From Jump Host (maintenance)                |

---

## 3. Детальний Опис Компонентів

### 3.1 Zone 0: Internet (Attacker Network)

#### 🎯 Kali Linux (192.168.125.100)
- **Роль:** Мережа атакуючого
- **Доступ:** Тільки до Frontend Zone через HTTP/HTTPS
- **Інструменти:**
  - Nmap для сканування
  - Burp Suite для web exploitation
  - Python скрипти для OCPP manipulation
  - SSH клієнт для pivoting

---

### 3.2 Zone 1: Frontend (Public DMZ)

#### 🖥️ Web Server (192.168.250.50)

**Технічні характеристики:**
- **OS:** Ubuntu 22.04 LTS
- **Stack:** Next.js 15.1.6 / React 19 / Node.js 20
- **Порти:** 80 (HTTP), 443 (HTTPS), 3000 (Next.js dev)

**Функціональність:**
- Публічний веб-сайт EcoCharge
- Користувацький кабінет (User Dashboard)
- Адміністративна панель (Admin Panel)
- API endpoints для взаємодії з API Gateway

**Вразливості:**

1. **CVE-2025-55182 - Next.js Image Optimization RCE**
   - Endpoint: `/api/image-proxy`
   - Вектор: Server-Side Request Forgery → Remote Code Execution
   - Результат: Reverse shell як `www-data`

2. **Privilege Escalation via Command Injection**
   - Endpoint: `/api/admin/backup`
   - Механізм: `sudo /opt/maintenance/backup.js` з інжекцією через `BACKUP_TARGET`
   - Результат: Root shell

3. **Information Disclosure**
   - Файл: `.env` містить `API_GATEWAY_URL` та `API_SECRET_KEY`
   - Файл: `/root/.ssh/id_jumphost` — SSH ключ до Jump Host

**FLAGS:**
- `FLAG{w3b_rCE_n3xtjs_1m4g3}` — після RCE
- `FLAG{pr1v3sc_b4ckup_sh3ll}` — після PrivEsc
- `FLAG{cr3d5_4nd_k3yz_1n_f1l3z}` — знахідка credentials

---

### 3.3 Zone 2: Management DMZ

#### ⚙️ API Gateway (192.168.100.20)

**Технічні характеристики:**
- **OS:** Ubuntu 22.04
- **Stack:** Node.js + Express
- **Порт:** 3000
- **Процес-менеджер:** PM2

**Функціональність:**
- Проксі до CitrineOS REST API
- Управління транзакціями зарядки
- Endpoints для веб-додатку

**Вразливості:**

1. **IDOR (Insecure Direct Object Reference)**
   - Endpoint: `/api/v1/user/transactions?user_id={ID}`
   - Вектор: Відсутність перевірки авторизації на user_id
   - Експлуатація: `user_id=1` повертає дані адміністратора

2. **Hardcoded API Key**
   - Ключ: `ec0ch4rg3_4p1_k3y_2024!`
   - Розташування: Витікає з `.env` файлу Web Server

**FLAG:**
- `FLAG{1d0r_tr4ns4ct10n_d4t4_l34k}` — при доступі до user_id=1

---

#### 📊 Grafana (192.168.100.30)

**Технічні характеристики:**
- **Версія:** Grafana 10.4.2
- **Порт:** 3000
- **Datasource:** PostgreSQL (CitrineOS DB)

**Функціональність:**
- Моніторинг інфраструктури EcoCharge
- Візуалізація даних CSMS
- Dashboard з метриками зарядних станцій

**Вразливості:**

1. **Default Credentials**
   - Login: `admin`
   - Password: `admin`
   - Відключено примусову зміну пароля

2. **Information Disclosure**
   - Dashboard "EcoCharge CSMS Overview" містить в description:
     ```
     DB Connection: postgresql://citrine:citrine@192.168.20.20:5432/citrine
     Hasura Admin Secret: CitrineOS!
     ```

**FLAG:**
- `FLAG{d3f4ult_gr4f4n4_cr3ds_l34k}` — в annotation дашборду

**Доступ:** Через SSH tunnel з Jump Host

---

#### 🔑 Jump Host (192.168.100.40 / 192.168.20.40 / 172.16.0.10)

**Технічні характеристики:**
- **OS:** Ubuntu 22.04
- **Роль:** Multi-homed bastion host
- **Мережеві інтерфейси:**
  - `eth0`: 192.168.100.40/24 (DMZ)
  - `eth1`: 192.168.20.40/24 (Internal)
  - `eth2`: 172.16.0.10/24 (OT)

**Функціональність:**
- SSH bastion для доступу до Internal та OT мереж
- Точка для network pivoting
- Інструменти для forensics та моніторингу

**Встановлені інструменти:**
- `tcpdump` — для sniffing OCPP трафіку
- `nmap` — для сканування мереж
- `postgresql-client` — для доступу до CSMS DB
- `wscat` / Python — для WebSocket (OCPP) взаємодії
- `curl`, `netcat`, `socat` — мережеві утиліти

**Вразливості:**
- SSH доступ з викраденим ключем `id_jumphost`
- Користувач: `operator`

**FLAG:**
- `FLAG{p1v0t_m4st3r_jumb0}` — в `/etc/motd` при SSH login

---

### 3.4 Zone 3: Internal Network

#### 💾 CitrineOS CSMS (192.168.20.20)

**Технічні характеристики:**
- **Platform:** Docker Compose
- **Компоненти:**
  - PostgreSQL 16 + PostGIS (порт 5432)
  - Hasura GraphQL Engine (порт 8180)
  - RabbitMQ (порт 5672, management 15672)
  - MinIO S3 Storage (порт 9000, console 9001)
  - Directus CMS (порт 8055)
  - CitrineOS Core

**Функціональність:**
- Charging Station Management System (CSMS)
- Управління зарядними станціями через OCPP
- База даних користувачів та транзакцій
- GraphQL API для інтеграцій

**OCPP Endpoints:**
- OCPP 1.6-J: `ws://192.168.20.20:8080`
- OCPP 2.0.1: `ws://192.168.20.20:8092`

**Credentials:**
- PostgreSQL: `citrine:citrine`
- RabbitMQ: `guest:guest`
- Hasura Admin Secret: `CitrineOS!`
- Directus: `admin@citrineos.com:CitrineOS!`

**Вразливості:**
- Доступ до PostgreSQL з Jump Host
- Credentials витікають через Grafana datasource

**FLAG:**
- `FLAG{db_dump_s3cr3t_t4bl3}` — в таблиці `ctf_flags`

---

### 3.5 Zone 4: OT Network (Operational Technology)

#### 🔌 EV Charger CP001 (172.16.0.40)

**Технічні характеристики:**
- **Platform:** EVerest (Docker)
- **Protocol:** OCPP 1.6-J
- **ConnectorId:** 1
- **CSMS Connection:** `ws://192.168.20.20:8080`

**Функціональність:**
- Емуляція зарядної станції AC (22kW)
- Підтримка основних OCPP команд
- Незашифрований WebSocket трафік

**FLAG:**
- `FLAG{0cpp_tr4ff1c_sn1ff3d}` — при sniffing OCPP трафіку

---

#### 🔌 EV Charger CP002 (172.16.0.60)

**Технічні характеристики:**
- **Platform:** EVerest (Docker)
- **Protocol:** OCPP 2.0.1
- **ConnectorId:** 1
- **CSMS Connection:** `ws://192.168.20.20:8092`

**Функціональність:**
- Емуляція зарядної станції AC (22kW)
- Підтримка OCPP 2.0.1 commands
- Вразливий до RemoteStopTransaction без авторизації

**FLAG:**
- `FLAG{0t_1mp4ct_ch4rg3r_st0pp3d}` — після відправки RemoteStopTransaction

---

## 4. Firewall конфігурація

### 4.1 Архітектура Firewall

Центральний firewall VM з 5 мережевими інтерфейсами керує трафіком між зонами через iptables.

### 4.2 Ключові правила (iptables)

```bash
# ============================================================================
# DEFAULT POLICIES
# ============================================================================
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# ============================================================================
# ZONE 0 → ZONE 1 (Internet → Frontend)
# ============================================================================
# Дозволити HTTP/HTTPS до Web Server
iptables -A FORWARD -i vmbr_internet -o vmbr_frontend \
    -d 192.168.250.50 -p tcp -m multiport --dports 80,443,3000 -j ACCEPT

# ============================================================================
# ZONE 1 → ZONE 2 (Frontend → DMZ)
# ============================================================================
# Web Server → API Gateway
iptables -A FORWARD -i vmbr_frontend -o vmbr_dmz \
    -s 192.168.250.50 -d 192.168.100.20 -p tcp --dport 3000 -j ACCEPT

# Web Server → Jump Host (SSH)
iptables -A FORWARD -i vmbr_frontend -o vmbr_dmz \
    -s 192.168.250.50 -d 192.168.100.40 -p tcp --dport 22 -j ACCEPT

# Web Server → Internal/OT: ЗАБОРОНЕНО
iptables -A FORWARD -i vmbr_frontend -o vmbr_internal -j DROP
iptables -A FORWARD -i vmbr_frontend -o vmbr_ot -j DROP

# ============================================================================
# ZONE 2 → ZONE 3 (DMZ → Internal)
# ============================================================================
# API Gateway → CitrineOS (все)
iptables -A FORWARD -i vmbr_dmz -o vmbr_internal \
    -s 192.168.100.20 -d 192.168.20.20 -j ACCEPT

# Grafana → PostgreSQL
iptables -A FORWARD -i vmbr_dmz -o vmbr_internal \
    -s 192.168.100.30 -d 192.168.20.20 -p tcp --dport 5432 -j ACCEPT

# Jump Host → Internal (повний доступ)
iptables -A FORWARD -i vmbr_dmz -o vmbr_internal \
    -s 192.168.100.40 -j ACCEPT

# ============================================================================
# ZONE 3 → ZONE 4 (Internal → OT)
# ============================================================================
# CSMS → Chargers (OCPP WebSocket)
iptables -A FORWARD -i vmbr_internal -o vmbr_ot \
    -s 192.168.20.20 -p tcp -m multiport --dports 8080,8092 -j ACCEPT

# ============================================================================
# ZONE 4 → ZONE 3 (OT → Internal)
# ============================================================================
# Chargers → CSMS (OCPP connect)
iptables -A FORWARD -i vmbr_ot -o vmbr_internal \
    -d 192.168.20.20 -p tcp -m multiport --dports 8080,8092 -j ACCEPT

# ============================================================================
# ZONE 2 → ZONE 4 (DMZ → OT) - ONLY JUMP HOST
# ============================================================================
# Jump Host → OT network
iptables -A FORWARD -i vmbr_dmz -o vmbr_ot \
    -s 192.168.100.40 -j ACCEPT

# ============================================================================
# ESTABLISHED CONNECTIONS
# ============================================================================
iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# ============================================================================
# LOGGING (для debugging)
# ============================================================================
iptables -A FORWARD -j LOG --log-prefix "FW-DROP: " --log-level 4
```

### 4.3 Таблиця дозволених з'єднань

| Source Zone | Destination Zone | Allowed Ports/Protocols | Purpose                  |
|-------------|------------------|-------------------------|--------------------------|
| Internet    | Frontend         | 80, 443, 3000 (TCP)     | HTTP/HTTPS до Web Server |
| Frontend    | DMZ (API GW)     | 3000 (TCP)              | API requests             |
| Frontend    | DMZ (Jump)       | 22 (TCP)                | SSH pivot                |
| DMZ (API)   | Internal (CSMS)  | All                     | API proxy                |
| DMZ (Grafana)| Internal (CSMS) | 5432 (TCP)              | PostgreSQL monitoring    |
| DMZ (Jump)  | Internal         | All                     | Admin access             |
| DMZ (Jump)  | OT               | All                     | Maintenance access       |
| Internal    | OT               | 8080, 8092 (TCP)        | OCPP WebSocket           |
| OT          | Internal         | 8080, 8092 (TCP)        | OCPP responses           |

---

## 5. Критичні залежності та потоки даних

### 5.1 OCPP Protocol Flow

```
CP001/CP002 <--WebSocket--> CitrineOS CSMS
    ↓                           ↓
OCPP 1.6/2.0.1            PostgreSQL DB
    ↓                           ↓
Unencrypted               Transaction logs
```

### 5.2 Attack Surface Summary

| Компонент        | Критичність | Вразливості                           | Impact          |
|------------------|-------------|---------------------------------------|-----------------|
| Web Server       | 🔴 Critical | RCE, PrivEsc, Info Disclosure         | Initial Access  |
| API Gateway      | 🟡 High     | IDOR, Hardcoded creds                 | Data Leak       |
| Grafana          | 🟡 High     | Default creds, Info Disclosure        | Credentials     |
| Jump Host        | 🟠 Medium   | Stolen SSH key                        | Network Pivot   |
| CitrineOS        | 🔴 Critical | Weak creds, No network isolation      | Data Exfiltration |
| Chargers (OT)    | 🔴 Critical | Unencrypted OCPP, No auth on commands | Physical Impact |

---

## 6. Схема атаки (High-Level)

```
[Attacker] 
    ↓ RCE
[Web Server] 
    ↓ PrivEsc → Credentials
[API Gateway] ← IDOR → FLAG #4
    ↓
[Jump Host] ← SSH Pivot → FLAG #6
    ↓ SSH Tunnel
[Grafana] ← Default Creds → FLAG #5 + DB Password
    ↓
[CSMS Database] ← psql → FLAG #7
    ↓
[OT Network] ← tcpdump → FLAG #8
    ↓
[Charger] ← RemoteStop → FLAG #9
```

---

## 7. Висновки

Інфраструктура EcoCharge демонструє типові помилки конфігурації та вразливості, які можуть призвести до повної компрометації системи:

1. **Недостатня сегментація мережі** — хоча архітектура має 5 зон, firewall правила дозволяють занадто широкий доступ
2. **Слабкі паролі та default credentials** — критична проблема для Grafana та CSMS
3. **Відсутність шифрування** — OCPP трафік передається у відкритому вигляді
4. **Lack of authentication на критичних endpoints** — RemoteStopTransaction без перевірки
5. **Information disclosure** — credentials витікають через конфігураційні файли

**Навчальна цінність:** Сценарій демонструє повний ланцюжок атаки від web exploitation до фізичного впливу на OT обладнання, що є критичним для розуміння захисту критичної інфраструктури.

---

**Автор:** Claude Assistant  
**Дата створення:** 9 лютого 2026  
**Для:** Дипломна робота - Cyber Range для EV Charging Infrastructure
