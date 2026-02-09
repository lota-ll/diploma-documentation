# EcoCharge Attack Flow - Презентаційна Діаграма

**Для друку / презентації науковому керівнику**

---

## Спрощена діаграма (для слайдів)

```mermaid
%%{init: {'theme':'base', 'themeVariables': { 'fontSize':'16px'}}}%%

flowchart TD
    Start([🎯 ПОЧАТОК АТАКИ<br/>Зовнішній атакуючий])
    
    subgraph Phase1[" 🔴 ФАЗА 1: Початковий доступ"]
        S1[1️⃣ RCE Exploit<br/>Next.js веб-сервер]
        S2[2️⃣ PrivEsc<br/>Отримання root]
        S3[3️⃣ Викрадення<br/>API ключів + SSH]
        S1 --> S2 --> S3
    end
    
    subgraph Phase2[" 🟡 ФАЗА 2: Латеральний рух"]
        S4[4️⃣ IDOR атака<br/>API Gateway]
        S5[5️⃣ SSH Pivot<br/>Jump Host]
        S4 --> S5
    end
    
    subgraph Phase3[" 🟢 ФАЗА 3: Внутрішня мережа"]
        S6[6️⃣ Grafana<br/>Default паролі]
        S7[7️⃣ Database Dump<br/>Витік даних CSMS]
        S6 --> S7
    end
    
    subgraph Phase4[" 🟣 ФАЗА 4: OT вплив"]
        S8[8️⃣ OCPP Sniffing<br/>Перехоплення трафіку]
        S9[9️⃣ ФІЗИЧНИЙ ВПЛИВ<br/>Зупинка зарядки]
        S8 --> S9
    end
    
    Start --> S1
    S3 --> S4
    S5 --> S6
    S7 --> S8
    S9 --> End([✅ ЦІЛЬ ДОСЯГНУТО<br/>9 FLAGS здобуто])
    
    style Start fill:#ff6b6b,stroke:#c92a2a,stroke-width:3px,color:#fff
    style End fill:#51cf66,stroke:#2f9e44,stroke-width:3px,color:#fff
    style Phase1 fill:#ffe0e0,stroke:#ff6b6b,stroke-width:2px
    style Phase2 fill:#fff4e0,stroke:#ffd43b,stroke-width:2px
    style Phase3 fill:#e0ffe0,stroke:#51cf66,stroke-width:2px
    style Phase4 fill:#f0e0ff,stroke:#9c36b5,stroke-width:2px
    
    style S1 fill:#ffc9c9
    style S2 fill:#ffc9c9
    style S3 fill:#ffc9c9
    style S4 fill:#ffe8a1
    style S5 fill:#ffe8a1
    style S6 fill:#b2f2bb
    style S7 fill:#b2f2bb
    style S8 fill:#e5dbff
    style S9 fill:#e5dbff,stroke:#862e9c,stroke-width:3px
```

---

## Мережева топологія (для схеми)

```mermaid
graph TB
    subgraph z0["🌐 ЗОВНІШНЯ МЕРЕЖА"]
        att[Атакуючий]
    end
    
    subgraph z1["🌍 ПУБЛІЧНА ЗОНА"]
        web[Веб-сервер<br/>Next.js]
    end
    
    subgraph z2["🛡️ DMZ"]
        api[API Gateway]
        graf[Grafana]
        jump[Jump Host]
    end
    
    subgraph z3["🗄️ ВНУТРІШНЯ МЕРЕЖА"]
        db[(База даних<br/>CSMS)]
    end
    
    subgraph z4["⚡ OT МЕРЕЖА"]
        ch1[Зарядна станція 1]
        ch2[Зарядна станція 2]
    end
    
    att -->|"①"| web
    web -->|"②③"| api
    web -.->|"④"| jump
    jump -->|"⑤"| graf
    graf -->|"⑥"| db
    jump -->|"⑦"| db
    jump -->|"⑧⑨"| ch1
    jump -->|"⑧⑨"| ch2
    
    classDef zone0 fill:#ffcccc,stroke:#cc0000
    classDef zone1 fill:#e3f2fd,stroke:#2196f3
    classDef zone2 fill:#fff9c4,stroke:#fbc02d
    classDef zone3 fill:#e8f5e9,stroke:#4caf50
    classDef zone4 fill:#f3e5f5,stroke:#9c27b0
    
    class att zone0
    class web zone1
    class api,graf,jump zone2
    class db zone3
    class ch1,ch2 zone4
```

---

## Ланцюжок вразливостей

```mermaid
graph LR
    V1[🔴 RCE<br/>CVE-2025-55182] --> V2[🔴 PrivEsc<br/>Command Injection]
    V2 --> V3[🟡 IDOR<br/>API Gateway]
    V3 --> V4[🟢 Default Creds<br/>Grafana]
    V4 --> V5[🟢 Weak Password<br/>Database]
    V5 --> V6[🟣 No Encryption<br/>OCPP]
    V6 --> V7[🟣 No Auth<br/>Stop Command]
    
    V7 --> I[💥 ФІЗИЧНИЙ<br/>ВПЛИВ]
    
    style V1 fill:#ff6b6b,color:#fff
    style V2 fill:#ff6b6b,color:#fff
    style V3 fill:#ffd43b
    style V4 fill:#51cf66
    style V5 fill:#51cf66
    style V6 fill:#9c36b5,color:#fff
    style V7 fill:#9c36b5,color:#fff
    style I fill:#e03131,color:#fff,stroke:#c92a2a,stroke-width:4px
```

---

## Статистика атаки

| Метрика | Значення |
|---------|----------|
| **Кількість фаз** | 4 |
| **Здобутих FLAGS** | 9 |
| **Компрометованих зон** | 5 з 5 (100%) |
| **Критичних вразливостей** | 7 |
| **Час атаки** | ~2.5 години |
| **Вплив** | Зупинка зарядної станції |

---

## Ключові висновки

### ✅ Продемонстровано:
1. Повний ланцюжок атаки від Web до OT
2. Реалістичні вразливості критичної інфраструктури
3. Важливість сегментації мережі
4. Вплив на фізичне обладнання

### ⚠️ Критичні проблеми:
1. **RCE у публічному веб-додатку** → Initial access
2. **Відсутність мережевої ізоляції** → Lateral movement
3. **Default/weak credentials** → Privilege escalation
4. **Незашифрований OCPP** → OT compromise
5. **Відсутність авторизації команд** → Physical impact

### 🎯 Навчальна цінність:
- Демонстрація Defense in Depth principles
- Практичне розуміння OT security
- Hands-on досвід з OCPP протоколом
- Red Team / Blue Team scenario

---

**Дата:** 9 лютого 2026  
**Версія:** 3.0  
**Призначення:** Дипломна робота - Cyber Range для електрозаправок
