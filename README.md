# 🛡️ siem-ai-dashboard

> An AI-powered, open-source SIEM dashboard for SOC teams —
> built on Wazuh, enriched with ML anomaly detection and LLM alert summarisation.

![Status](https://img.shields.io/badge/status-in%20development-orange)
![License](https://img.shields.io/badge/license-MIT-blue)
![Stack](https://img.shields.io/badge/stack-Wazuh%20%7C%20OpenSearch%20%7C%20Python-informational)
![Cost](https://img.shields.io/badge/license%20cost-%240-brightgreen)

---

## 📌 Overview

**siem-ai-dashboard** is a proof-of-concept AI-enhanced Security Information and Event Management (SIEM) system built entirely on open-source tools. It extends Wazuh's native threat detection capabilities with a custom Python AI layer that scores alerts for anomalies, profiles user behaviour, and generates plain-English summaries for SOC analysts — reducing triage time and alert fatigue.

Unlike commercial SIEM solutions, this stack costs $0 in licensing and is fully customisable. It is designed for SOC teams, security engineers, and students who want a production-grade detection and response workflow without vendor lock-in.

---

## 🎯 Goals

1. ✅ Reduce alert fatigue for L1 analysts via AI-driven severity scoring
2. ✅ Surface plain-English alert summaries using LLMs (Claude / OpenAI)
3. ✅ Detect common attack patterns mapped to MITRE ATT&CK framework
4. ✅ Automate tier-1 response actions — IP blocking, incident ticket creation
5. ✅ Enrich alerts automatically with threat intelligence (VirusTotal, GeoIP)
6. ✅ Provide a fully open-source, zero-cost alternative to commercial SIEMs

---

## 🏗️ Architecture

```
Log Sources (endpoints, firewalls, cloud, apps)
        ↓
Wazuh Agent  ──────────────────────────────────────
        ↓                                          │
Wazuh Manager (correlation + rule engine)          │
        ↓                                          │
Wazuh Indexer — OpenSearch (log storage)           │
        ↓                                          │
Python AI Microservice (FastAPI)                   │
  ├── Isolation Forest — anomaly scoring           │
  ├── UEBA — user behaviour baseline               │
  └── LLM Summariser — plain-English alerts        │
        ↓                                          │
Custom OpenSearch Dashboard (SOC UI)  ─────────────
        ↓
Integrations
  ├── TheHive      — incident case management
  ├── Shuffle      — SOAR automation
  └── VirusTotal   — threat intel enrichment
```

---

## 🛠️ Tech Stack

| Layer | Tools |
|---|---|
| **SIEM Core** | Wazuh, OpenSearch, OpenSearch Dashboards |
| **AI / ML** | Python 3.11, scikit-learn, FastAPI, Claude API / OpenAI API |
| **SOC Workflow** | TheHive, Shuffle SOAR, VirusTotal API |
| **Infra** | Docker Compose, Ubuntu 22.04 |
| **Testing** | Atomic Red Team, Caldera |
| **Docs** | Markdown, Excalidraw |

---

## ✅ Use Cases (20 total)

| Category | Examples |
|---|---|
| 🔴 Threat detection | Brute force, lateral movement, privilege escalation, ransomware FIM, data exfil |
| 🟡 AI anomaly detection | Unusual login time, abnormal data volume, new device/IP per user |
| 🟢 SOC automation | Auto incident ticket, LLM alert summary, auto IP block, hash lookup |
| 🔵 Dashboard visibility | Real-time alert queue, MITRE heatmap, geo attack map, analyst KPIs |

→ Full list in [`docs/03-use-cases.md`](docs/03-use-cases.md)

---

## ⚡ Quick Start

### Requirements

| Spec | Minimum | Recommended |
|---|---|---|
| RAM | 8 GB (Wazuh only) | 16 GB (full stack) |
| Disk | 60 GB free | 80 GB |
| OS | Ubuntu 22.04 / WSL2 | Ubuntu 22.04 |
| Docker | 24.0+ | latest |

### 1. Clone the repo

```bash
git clone https://github.com/berryliciousbunny/siem-ai-dashboard.git
cd siem-ai-dashboard
```

### 2. Set up environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in your API keys:

```env
OPENAI_API_KEY=your_key_here        # or ANTHROPIC_API_KEY
VIRUSTOTAL_API_KEY=your_key_here
THEHIVE_URL=http://localhost:9000
THEHIVE_API_KEY=your_key_here
```

### 3. Start the full stack

```bash
docker compose up -d
```

This starts: Wazuh Manager, Wazuh Indexer, Wazuh Dashboard, TheHive, Shuffle, and the Python AI microservice.

### 4. Access the dashboard

| Service | URL | Default credentials |
|---|---|---|
| Wazuh Dashboard | http://localhost:5601 | admin / SecretPassword |
| TheHive | http://localhost:9000 | admin@thehive.local / secret |
| Shuffle | http://localhost:3001 | set on first login |
| AI API (FastAPI) | http://localhost:8000/docs | — |

### 5. Install a Wazuh agent (on an endpoint to monitor)

```bash
# On the endpoint machine — replace WAZUH_MANAGER_IP with Laptop B's IP
curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.0-1_amd64.deb \
  && sudo WAZUH_MANAGER=WAZUH_MANAGER_IP dpkg -i ./wazuh-agent.deb
sudo systemctl start wazuh-agent
```

Once the agent is running, logs will start flowing into the dashboard within a few minutes.

---

## 📁 Project Structure

```
siem-ai-dashboard/
├── README.md
├── CONTRIBUTING.md
├── docker-compose.yml
├── .env.example
│
├── docs/
│   ├── 01-overview.md
│   ├── 02-architecture.md
│   ├── 03-use-cases.md
│   ├── 04-deployment.md
│   ├── 05-ai-layer.md
│   ├── 06-roadmap.md
│   └── 07-test-results.md
│
├── diagrams/
│   ├── architecture.excalidraw
│   └── architecture.png
│
├── rules/
│   ├── brute_force.xml
│   ├── lateral_movement.xml
│   ├── ransomware_fim.xml
│   └── privilege_escalation.xml
│
├── dashboards/
│   ├── soc-analyst-view.json
│   └── ciso-summary-view.json
│
└── ai/
    ├── main.py
    ├── anomaly.py
    ├── ueba.py
    ├── summarizer.py
    ├── requirements.txt
    └── Dockerfile
```

---

## 🗺️ Roadmap

| Phase | Status |
|---|---|
| Phase 1 — Documentation & PoC planning | 🚧 In progress |
| Phase 2 — Wazuh deployment | ⬜ Planned |
| Phase 3 — Custom detection rules | ⬜ Planned |
| Phase 4 — Custom OpenSearch dashboard | ⬜ Planned |
| Phase 5 — AI / ML layer | ⬜ Planned |
| Phase 6 — Integrations | ⬜ Planned |
| Phase 7 — Testing & red team simulation | ⬜ Planned |
| Phase 8 — Finalise & publish | ⬜ Planned |

→ Full details in [`docs/06-roadmap.md`](docs/06-roadmap.md)

---

## 🤝 Contributing

Contributions are welcome — new detection rules, dashboard panels, AI model improvements, or documentation fixes.

→ See [`CONTRIBUTING.md`](CONTRIBUTING.md) for guidelines.

---

## 📄 License

MIT License — free to use, modify, and distribute.

---

> Built as a proof of concept for AI-enhanced SOC operations.
> Open-source stack — $0 licensing cost.