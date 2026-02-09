# EcoCharge Cyber Range - Network Topology (SHORT)

**Версія:** short

---

## Мережева архітектура

**5 зон безпеки:**

```
Internet (192.168.125.0/24)
    ↓
Frontend (192.168.250.0/24) - Web Server
    ↓
Management DMZ (192.168.100.0/24) - API, Grafana, Jump Host
    ↓
Internal (192.168.20.0/24) - CSMS Database
    ↓
OT Network (172.16.0.0/24) - Зарядні станції
```

---

## Компоненти

### Zone 1: Frontend
- **Web Server (192.168.250.50)** - Next.js, RCE + PrivEsc, FLAGS #1-3

### Zone 2: DMZ
- **API Gateway (192.168.100.20)** - IDOR, FLAG #4
- **Grafana (192.168.100.30)** - Default creds, FLAG #5
- **Jump Host (192.168.100.40)** - Multi-homed bastion, FLAG #6

### Zone 3: Internal
- **CitrineOS (192.168.20.20)** - PostgreSQL + CSMS, FLAG #7

### Zone 4: OT
- **CP001 (172.16.0.40)** - OCPP 1.6-J, FLAG #8
- **CP002 (172.16.0.60)** - OCPP 2.0.1, FLAG #9

---

## Firewall Rules

| From | To | Allow | Block |
|------|----|----|-------|
| Internet | Frontend | HTTP/HTTPS | All else |
| Frontend | DMZ | API (3000), SSH (22) | Internal, OT |
| DMZ | Internal | Specific ports | - |
| DMZ | OT | Jump Host only | Direct access |

---

## Вразливості

1. **Web:** CVE-2025-55182 RCE, PrivEsc
2. **API:** IDOR, Hardcoded credentials  
3. **Grafana:** admin:admin, Info disclosure
4. **CSMS:** Weak passwords, No encryption
5. **OT:** Plaintext OCPP, No auth on commands

---

## Attack Surface

**Critical Path:**  
RCE → PrivEsc → IDOR → Pivot → DB Dump → OCPP Sniff → Physical Impact

**9 FLAGS** демонструють повний ланцюжок від web до OT.
