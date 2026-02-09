# EcoCharge Attack Flow - Візуальна Діаграма

**Призначення:** Презентація для наукового керівника  
**Формат:** Mermaid діаграма (рендериться в GitHub, Obsidian, VS Code)

---

## Attack Flow Diagram

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'primaryColor':'#ffcccc','primaryTextColor':'#000','primaryBorderColor':'#cc0000','lineColor':'#333','secondaryColor':'#e3f2fd','tertiaryColor':'#fff9c4'}}}%%

graph TB
    %% Styling
    classDef phase1 fill:#ffcccc,stroke:#cc0000,stroke-width:3px,color:#000
    classDef phase2 fill:#fff9c4,stroke:#fbc02d,stroke-width:3px,color:#000
    classDef phase3 fill:#e8f5e9,stroke:#4caf50,stroke-width:3px,color:#000
    classDef phase4 fill:#f3e5f5,stroke:#9c27b0,stroke-width:3px,color:#000
    classDef flag fill:#ffd54f,stroke:#f57c00,stroke-width:2px,color:#000
    
    %% PHASE 1: Initial Access
    Start([🎯 Attacker<br/>Kali Linux]):::phase1
    
    Start -->|"1. Nmap Scan"| Recon[🔍 Reconnaissance<br/>192.168.250.50<br/>Next.js 15 detected]:::phase1
    Recon -->|"2. CVE-2025-55182"| RCE[💥 Remote Code Execution<br/>/api/image-proxy<br/>Shell: www-data]:::phase1
    RCE --> Flag1[🚩 FLAG #1<br/>w3b_rCE_n3xtjs_1m4g3]:::flag
    
    RCE -->|"3. Command Injection"| PrivEsc[⬆️ Privilege Escalation<br/>backup.js sudo<br/>Shell: root]:::phase1
    PrivEsc --> Flag2[🚩 FLAG #2<br/>pr1v3sc_b4ckup_sh3ll]:::flag
    
    PrivEsc -->|"4. File Discovery"| Loot[📂 Credential Loot<br/>.env file<br/>SSH key: id_jumphost]:::phase1
    Loot --> Flag3[🚩 FLAG #3<br/>cr3d5_4nd_k3yz_1n_f1l3z]:::flag
    
    %% PHASE 2: Lateral Movement
    Loot -->|"5. API Request<br/>user_id=1"| IDOR[🔓 IDOR Exploitation<br/>API Gateway<br/>192.168.100.20]:::phase2
    IDOR --> Flag4[🚩 FLAG #4<br/>1d0r_tr4ns4ct10n_d4t4]:::flag
    
    Loot -->|"6. SSH Pivot<br/>stolen key"| Jump[🔑 Jump Host Access<br/>operator@192.168.100.40<br/>Multi-homed bastion]:::phase2
    Jump --> Flag6[🚩 FLAG #6<br/>p1v0t_m4st3r_jumb0]:::flag
    
    %% PHASE 3: Internal Compromise
    Jump -->|"7. SSH Tunnel<br/>:3000"| Grafana[📊 Grafana Dashboard<br/>admin:admin<br/>DB creds leaked]:::phase3
    Grafana --> Flag5[🚩 FLAG #5<br/>d3f4ult_gr4f4n4_cr3ds]:::flag
    
    Grafana -->|"8. PostgreSQL<br/>citrine:citrine"| Database[💾 Database Dump<br/>CitrineOS CSMS<br/>192.168.20.20]:::phase3
    Database --> Flag7[🚩 FLAG #7<br/>db_dump_s3cr3t_t4bl3]:::flag
    
    %% PHASE 4: OT Impact
    Jump -->|"9. tcpdump<br/>eth2"| Sniff[📡 OCPP Sniffing<br/>Plaintext WebSocket<br/>172.16.0.0/24]:::phase4
    Sniff --> Flag8[🚩 FLAG #8<br/>0cpp_tr4ff1c_sn1ff3d]:::flag
    
    Sniff -->|"10. RemoteStop<br/>Transaction"| Impact[⚡ Physical Impact<br/>Charger CP002<br/>Service Disruption]:::phase4
    Impact --> Flag9[🚩 FLAG #9<br/>0t_1mp4ct_ch4rg3r_st0pp3d]:::flag
    
    %% Victory
    Flag9 --> Victory([🎉 Complete Compromise<br/>Web → OT]):::phase4
    
    %% Phase Labels
    subgraph P1[" "]
        direction TB
        Start
        Recon
        RCE
        Flag1
        PrivEsc
        Flag2
        Loot
        Flag3
    end
    
    subgraph P2[" "]
        direction TB
        IDOR
        Flag4
        Jump
        Flag6
    end
    
    subgraph P3[" "]
        direction TB
        Grafana
        Flag5
        Database
        Flag7
    end
    
    subgraph P4[" "]
        direction TB
        Sniff
        Flag8
        Impact
        Flag9
        Victory
    end
    
    %% Phase Headers (using notes)
    P1 -.->|PHASE 1:<br/>Initial Access| P2
    P2 -.->|PHASE 2:<br/>Lateral Movement| P3
    P3 -.->|PHASE 3:<br/>Internal Access| P4
    
    style P1 fill:#fff0f0,stroke:#cc0000,stroke-width:2px
    style P2 fill:#fffef0,stroke:#fbc02d,stroke-width:2px
    style P3 fill:#f0fff0,stroke:#4caf50,stroke-width:2px
    style P4 fill:#f8f0ff,stroke:#9c27b0,stroke-width:2px
```

---

## Attack Timeline

```mermaid
gantt
    title EcoCharge Attack Progression
    dateFormat HH:mm
    axisFormat %H:%M
    
    section Phase 1
    Reconnaissance       :done, p1a, 10:00, 10:15
    RCE Exploitation     :done, p1b, 10:15, 10:30
    Privilege Escalation :done, p1c, 10:30, 10:45
    Credential Discovery :done, p1d, 10:45, 11:00
    
    section Phase 2
    IDOR Attack         :done, p2a, 11:00, 11:15
    SSH Pivoting        :done, p2b, 11:15, 11:30
    
    section Phase 3
    Grafana Access      :done, p3a, 11:30, 11:45
    Database Dump       :done, p3b, 11:45, 12:00
    
    section Phase 4
    OCPP Sniffing       :done, p4a, 12:00, 12:20
    Physical Impact     :crit, p4b, 12:20, 12:30
```

---

## Network Flow Diagram

```mermaid
graph LR
    %% Zones
    subgraph Internet["🌐 INTERNET<br/>192.168.125.0/24"]
        A[Attacker]
    end
    
    subgraph Frontend["🌍 FRONTEND<br/>192.168.250.0/24"]
        W[Web Server<br/>:80,:443,:3000]
    end
    
    subgraph DMZ["🛡️ MANAGEMENT DMZ<br/>192.168.100.0/24"]
        API[API Gateway<br/>:3000]
        G[Grafana<br/>:3000]
        J[Jump Host<br/>Multi-homed]
    end
    
    subgraph Internal["🗄️ INTERNAL<br/>192.168.20.0/24"]
        DB[(CSMS DB<br/>:5432)]
    end
    
    subgraph OT["⚡ OT NETWORK<br/>172.16.0.0/24"]
        C1[CP001<br/>OCPP 1.6]
        C2[CP002<br/>OCPP 2.0.1]
    end
    
    %% Connections
    A -->|"① RCE"| W
    W -->|"② IDOR"| API
    W -.->|"③ SSH"| J
    J -->|"④ Tunnel"| G
    G -->|"⑤ SQL"| DB
    API -->|GraphQL| DB
    J -->|"⑥ psql"| DB
    J -->|"⑦ Sniff"| C1
    J -->|"⑧ Stop"| C2
    C1 & C2 -->|OCPP| DB
    
    %% Styling
    classDef internet fill:#ffcccc,stroke:#cc0000,stroke-width:2px
    classDef frontend fill:#e3f2fd,stroke:#2196f3,stroke-width:2px
    classDef dmz fill:#fff9c4,stroke:#fbc02d,stroke-width:2px
    classDef internal fill:#e8f5e9,stroke:#4caf50,stroke-width:2px
    classDef ot fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px
    
    class A internet
    class W frontend
    class API,G,J dmz
    class DB internal
    class C1,C2 ot
```

---

## Vulnerability Chain

```mermaid
flowchart TD
    V1[CVE-2025-55182<br/>Next.js RCE] -->|Exploit| V2[Command Injection<br/>backup.js PrivEsc]
    V2 -->|Credentials| V3[IDOR<br/>API Gateway]
    V3 -->|Access| V4[Default Credentials<br/>Grafana admin:admin]
    V4 -->|Database Creds| V5[Weak Passwords<br/>PostgreSQL citrine:citrine]
    V5 -->|OT Access| V6[Unencrypted OCPP<br/>Plaintext WebSocket]
    V6 -->|Commands| V7[No Authorization<br/>RemoteStopTransaction]
    
    V7 --> Impact[💥 Physical Impact<br/>Charger Stopped]
    
    style V1 fill:#ff8a80
    style V2 fill:#ff8a80
    style V3 fill:#ffab91
    style V4 fill:#ffcc80
    style V5 fill:#fff59d
    style V6 fill:#ffab91
    style V7 fill:#ff8a80
    style Impact fill:#f44336,color:#fff,stroke:#b71c1c,stroke-width:3px
```

---

## Impact Pyramid

```mermaid
graph BT
    L1[Web Server Compromise]
    L2[DMZ Network Access]
    L3[Internal Database Breach]
    L4[OT Network Infiltration]
    L5[⚠️ PHYSICAL IMPACT ⚠️<br/>Service Disruption]
    
    L1 --> L2
    L2 --> L3
    L3 --> L4
    L4 --> L5
    
    style L1 fill:#e3f2fd,stroke:#2196f3
    style L2 fill:#fff9c4,stroke:#fbc02d
    style L3 fill:#e8f5e9,stroke:#4caf50
    style L4 fill:#f3e5f5,stroke:#9c27b0
    style L5 fill:#ffcdd2,stroke:#d32f2f,stroke-width:4px
```

---

## Технічна інформація

**Автор:** Claude Assistant  
**Дата:** 9 лютого 2026  
**Призначення:** Дипломна робота - Cyber Range для захисту інфраструктури електрозаправок  
**Формат:** Mermaid.js (сумісно з GitHub, GitLab, Obsidian, VS Code)

### Як використовувати:

1. **GitHub/GitLab:** Скопіювати мермаїд блоки в .md файл - автоматично рендеряться
2. **PowerPoint:** Використати [Mermaid Chart](https://www.mermaidchart.com/) для експорту в PNG/SVG
3. **LaTeX:** Конвертувати в PDF через pandoc або mermaid-cli
4. **Презентація:** Імпортувати SVG у PowerPoint/Google Slides

### Команди для експорту:

```bash
# Встановити mermaid-cli
npm install -g @mermaid-js/mermaid-cli

# Конвертувати в PNG (висока якість)
mmdc -i attack_flow_diagram.md -o attack_flow.png -w 2400 -H 1800 -b transparent

# Конвертувати в SVG (векторна графіка)
mmdc -i attack_flow_diagram.md -o attack_flow.svg -b transparent

# Конвертувати в PDF
mmdc -i attack_flow_diagram.md -o attack_flow.pdf
```
