# EcoCharge CTF - Attack Flow (SHORT)

**Версія:** short

---

## Сценарій: Web → OT (9 FLAGS)

---

### PHASE 1: Initial Access 🎯

**Target:** Web Server (192.168.250.50)

1. **RCE** → CVE-2025-55182 (Next.js) → `FLAG #1`
2. **PrivEsc** → backup.js injection → `FLAG #2`
3. **Loot** → .env + SSH key → `FLAG #3`

---

### PHASE 2: Lateral Movement 🎯

**Target:** DMZ (192.168.100.0/24)

4. **IDOR** → API Gateway user_id=1 → `FLAG #4`
5. **SSH Pivot** → Jump Host (stolen key) → `FLAG #6`

---

### PHASE 3: Internal Compromise 🎯

**Target:** CSMS (192.168.20.20)

6. **Grafana** → admin:admin + SSH tunnel → `FLAG #5`
7. **Database** → psql dump (leaked creds) → `FLAG #7`

---

### PHASE 4: OT Impact 🎯

**Target:** Chargers (172.16.0.0/24)

8. **Sniffing** → tcpdump OCPP traffic → `FLAG #8`
9. **Physical** → RemoteStopTransaction → `FLAG #9`

---

## Kill Chain

```
RCE → PrivEsc → Credentials → IDOR → Pivot → 
Grafana → Database → OCPP Sniff → Charger Stop
```

---

## Attack Summary

| Phase | Vector | Impact | Flags |
|-------|--------|--------|-------|
| 1 | Web Exploitation | Code Execution, Root Access | #1, #2, #3 |
| 2 | Network Pivot | DMZ Access | #4, #6 |
| 3 | Data Exfiltration | Database Compromise | #5, #7 |
| 4 | OT Impact | Physical Damage | #8, #9 |

---

## Tools

**Attacker:** Kali Linux, nmap, curl, tcpdump, wscat  
**Victim:** Next.js, Node.js, Grafana, PostgreSQL, EVerest

---

## Required Skills

- Web exploitation (RCE, PrivEsc)
- API security (IDOR)
- Network pivoting (SSH tunneling)
- Database (PostgreSQL)
- OT protocols (OCPP)

---

## Key Vulnerabilities

1. CVE-2025-55182 - Next.js RCE
2. Command Injection - backup.js
3. IDOR - API Gateway
4. Default Credentials - Grafana
5. Unencrypted OCPP - Chargers
6. No Command Authorization - CSMS

---

**Результат:** Повна компрометація від публічного веб-сайту до фізичної зупинки зарядної станції.
