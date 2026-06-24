# Hybrid Homelab + AI Agents

A hybrid homelab simulating a real corporate environment, running across two physical machines with AI agents acting as employees. 
---

## Objective

Simulate a small company IT infrastructure where:
- **AI agents** perform daily tasks as employees (login, tickets, navigation).
- **Chaos Monkey** sabotages the environment in controlled ways.
- Generate realistic troubleshooting scenarios that are investigated, resolved, and documented.

---

## Hardware

| Machine | Role | Specs |
|---|---|---|
| PC1 — Samsung | Proxmox Server | i5-7200U · 16GB RAM |
| PC2 — HP      | VMware Workstation | i5-8265U · 24GB RAM |

---

## Architecture

```
[ ISP / Home Router ]
        |
      vmbr0 (physical NIC)
        |
   OPNsense VM (PC1)       ← firewall / gateway
        |
      vmbr1 (internal bridge)
        |
   192.168.30.0/24  ←  office.lab
        |
   ┌────────────────────────────────┐
   │  PC1 — Proxmox                 │
   │  ├── DC01 (AD/DNS/DHCP)        │
   │  ├── SRV-MON (Zabbix/Grafana)  │
   │  └── SRV-APPS (GLPI + AI)      │
   └────────────────────────────────┘
        |
   ┌────────────────────────────────┐
   │  PC2 — VMware Workstation      │
   │  ├── PC01 — Windows 11         │
   │  ├── PC02 — Ubuntu Desktop     │
   │  ├── PC03 — Ubuntu Desktop     │
   │  └── PC04 — Ubuntu Desktop     │
   └────────────────────────────────┘
```

---

## Stack

| Layer | Technology |
|---|---|
| Firewall / Gateway | OPNsense 26.1.10 |
| Domain Controller | Windows Server 2022 Core (AD DS, DNS, DHCP) |
| Cloud Identity | Microsoft Entra ID (hybrid sync) |
| Endpoint Management | Microsoft Intune |
| Monitoring | Zabbix + Grafana |
| Service Desk | GLPI |
| Automation | Ansible |
| AI Agents | Python + Claude API |
| Chaos Engineering | Chaos Monkey (custom scripts) |
| Documentation | Cisco Packet Tracer + GitHub |

---

## Network

| Device | IP | Role |
|---|---|---|
| OPNsense LAN | 192.168.30.1 | Gateway (OPNsense) |
| DC01 | 192.168.30.10 | Domain Controller |
| SRV-MON | 192.168.30.20 | Monitoring |
| SRV-APPS | 192.168.30.30 | Apps + AI Agents |

DHCP Pool: 192.168.30.100 - 192.168.30.200

| PC01 | 192.168.30.101 | Windows 11 client |
| PC02 | 192.168.30.102 | Ubuntu Desktop client |
| PC03 | 192.168.30.103 | Ubuntu Desktop client |
| PC04 | 192.168.30.104 | Ubuntu Desktop client |

---

## Repository Structure

```
hybrid-homelab
├── README.md
├── 01-infrastructure
│   ├── proxmox
│   ├── opnsense
│   └── dc01
├── 02-monitoring
│   ├── zabbix
│   └── grafana
├── 03-applications
│   ├── glpi
│   └── ai-agents
├── 04-cloud
│   ├── entra-id
│   └── intune
├── 05-automation
│   ├── ansible
│   └── chaos-monkey
└── diagrams
```

---

## Roadmap

- [x] Proxmox VE 9.1.11 installed and configured
- [x] OPNsense 26.1.10 installed (WAN + LAN)
- [ ] DC01 — Windows Server 2022 Core (AD DS, DNS, DHCP)
- [ ] SRV-MON — Zabbix + Grafana
- [ ] SRV-APPS — GLPI + AI Agents
- [ ] PC01/02/03/04 — joined to domain
- [ ] Entra ID hybrid sync + Intune enrollment
- [ ] AI agents simulating employees
- [ ] Chaos Monkey scenarios + troubleshooting documentation

---

