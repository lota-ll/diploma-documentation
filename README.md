# EcoCharge CTF - Scenario Documentation

## Overview

This repository contains documentation for the EcoCharge CTF (Capture The Flag) scenario - a cybersecurity training environment simulating attacks on Electric Vehicle Charging Station Management Systems (CSMS).


## Documentation Structure

```
diploma-documentation-v4/
├── README.md                           # This file
├── scenario_attack_flow_v4_SHORT.md    # Quick reference attack guide
├── scenario_topology_v4_SHORT.md       # Quick reference network diagram
└── full_documentation/
    ├── scenario_attack_flow_v4_FULL.md # Detailed attack walkthrough
    └── scenario_topology_v4_FULL.md    # Detailed infrastructure docs
```

## Quick Start

### For CTF Players:
1. Start with `scenario_topology_v4_SHORT.md` to understand the network
2. Follow `scenario_attack_flow_v4_SHORT.md` for attack steps
3. Reference full documentation if stuck

### For CTF Organizers:
1. Review full documentation for deployment details
2. Check firewall rules in topology documentation
3. Verify all flags are properly placed

## Flags Overview

| Phase | Flag | Difficulty |
|-------|------|------------|
| 1 | Command Injection RCE | ⭐⭐ |
| 1 | Privilege Escalation | ⭐⭐⭐ |
| 1 | Credential Discovery | ⭐ |
| 2 | API Info Disclosure | ⭐⭐ |
| 2 | SSH Pivot | ⭐⭐ |
| 3 | Grafana Default Creds | ⭐ |
| 4 | CVE-2025-55182 RCE | ⭐⭐⭐⭐ |
| 4 | Environment Leak | ⭐⭐⭐ |
| 4 | Database Compromise | ⭐⭐⭐ |

**Total Flags:** 9  
**Estimated Time:** 4-8 hours

## Key Vulnerabilities

### 1. CWE-78: Command Injection (Web Server)
- **Location:** `/api/qr` endpoint
- **Parameter:** `station`
- **Discovery:** Debug info disclosure with invalid format

### 2. CVE-2025-55182: React2Shell (CSMS)
- **Affected:** Next.js 15.1.2 + React 19.x
- **Type:** Unsafe Deserialization → Pre-auth RCE
- **CVSS:** 10.0 (Critical)

## Related Repositories

- `ecocharge-website` - Vulnerable web server code
- `api-gateway` - DMZ API gateway
- `grafana-deploy` - Grafana configuration
- `jumphost-config` - Jump host setup
- `citrineos-deploy` - CSMS deployment

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 4.0 | Feb 2025 | CWE-78 initial access, CVE-2025-55182 for CSMS |
| 3.0 | Jan 2025 | CVE-2025-55182 for initial access |
| 2.0 | Dec 2024 | Added DMZ components |
| 1.0 | Nov 2024 | Initial scenario |

## License

Educational use only. Created for cybersecurity training purposes.

---

**Author:** Diploma Project  
**Last Updated:** February 2025
