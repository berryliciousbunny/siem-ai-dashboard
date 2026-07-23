# 02 — Architecture

> This document covers the full system design, component breakdown, data flow, network reference, and infrastructure requirements. For project context see [`01-overview.md`](01-overview.md). For deployment steps see [`04-deployment.md`](04-deployment.md).

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [System Diagram](#system-diagram)
- [Component Breakdown](#component-breakdown)
- [Data Flow — Step by Step](#data-flow--step-by-step)
- [Network & Ports Reference](#network--ports-reference)
- [Infrastructure Requirements](#infrastructure-requirements)

---

## Architecture Overview

The system is built in four distinct layers. The **collection layer** gathers raw logs from monitored endpoints and network devices via Wazuh agents and syslog forwarders. The **detection layer** — Wazuh Manager and its rule engine — correlates incoming events against a library of custom and built-in detection rules, generating structured alerts. The **AI enrichment layer** — a Python FastAPI microservice — consumes those alerts in real time, scores them for statistical anomaly, checks them against per-user behavioural baselines, and appends a plain-English LLM summary before writing the enriched alert back to storage. The **presentation and response layer** surfaces everything through a custom OpenSearch dashboard for SOC analysts while simultaneously triggering automated responses via TheHive, Shuffle SOAR, and Wazuh active response scripts.

All components run as Docker containers orchestrated by a single `docker-compose.yml`, making the full stack deployable on a single machine with one command.

---

## System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        MONITORED ENDPOINTS                       │
│         Windows Hosts · Linux Servers · Network Devices          │
└───────────────────────────┬─────────────────────────────────────┘
                            │  Wazuh Agent (installed on each host)
                            │  Syslog / API log forwarders
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        WAZUH MANAGER                             │
│                                                                   │
│  ┌─────────────────┐   ┌──────────────────┐   ┌──────────────┐  │
│  │  Log Decoder    │ → │  Rule Engine     │ → │  Alert Queue │  │
│  │  (normalise)    │   │  (correlate)     │   │  (OpenSearch)│  │
│  └─────────────────┘   └──────────────────┘   └──────┬───────┘  │
│                                                        │          │
│  Custom rules: rules/brute_force.xml                  │          │
│                rules/lateral_movement.xml             │          │
│                rules/ransomware_fim.xml               │          │
│                rules/privilege_escalation.xml         │          │
└────────────────────────────────────────────────────────┼─────────┘
                                                         │
                                                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                     WAZUH INDEXER (OpenSearch)                   │
│                                                                   │
│   wazuh-alerts-*          wazuh-ai-enriched-*                    │
│   (raw alerts)            (AI scored + LLM summary)              │
└───────────┬──────────────────────────┬──────────────────────────┘
            │                          │
            ▼                          ▼
┌───────────────────────┐   ┌──────────────────────────────────────┐
│   PYTHON AI SERVICE   │   │       WAZUH DASHBOARD                │
│   (FastAPI :8000)     │   │       (OpenSearch Dashboards :5601)  │
│                       │   │                                       │
│  ┌─────────────────┐  │   │  ┌────────────────────────────────┐  │
│  │ Isolation Forest│  │   │  │ Real-time Alert Queue (UC-017) │  │
│  │ anomaly scorer  │  │   │  │ MITRE ATT&CK Heatmap  (UC-018) │  │
│  └────────┬────────┘  │   │  │ Geo Attack Map        (UC-019) │  │
│  ┌────────▼────────┐  │   │  │ Analyst KPI Panel     (UC-020) │  │
│  │ UEBA baseline   │  │   │  └────────────────────────────────┘  │
│  │ per-user model  │  │   └──────────────────────────────────────┘
│  └────────┬────────┘  │
│  ┌────────▼────────┐  │
│  │ LLM Summariser  │──┼──→  Claude API / OpenAI API
│  │ (Claude/OpenAI) │  │
│  └────────┬────────┘  │
└───────────┼───────────┘
            │  Enriched alert written back to OpenSearch
            │  Integrations triggered in parallel
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    INTEGRATIONS LAYER                            │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │   TheHive    │  │   Shuffle    │  │  VirusTotal API       │  │
│  │   :9000      │  │   SOAR :3001 │  │  (external)           │  │
│  │              │  │              │  │                        │  │
│  │ Auto incident│  │ Auto IP block│  │ File hash lookup       │  │
│  │ ticket       │  │ Slack notify │  │ Domain reputation      │  │
│  └──────────────┘  └──────────────┘  └───────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Breakdown

### Wazuh Agent

| Field | Detail |
|---|---|
| **What it is** | Lightweight endpoint agent installed on each monitored host |
| **Role in stack** | Collects logs, monitors file integrity, tracks running processes, and forwards everything to the Wazuh Manager |
| **Supported platforms** | Windows, Linux, macOS |
| **Communication** | Sends encrypted data to Wazuh Manager on port `1514` (UDP/TCP) |
| **Config location** | `/var/ossec/etc/ossec.conf` on each agent host |

Key capabilities used in this project:
- Log collection (auth logs, Windows Event Log, Sysmon)
- File Integrity Monitoring (FIM) — used for UC-005 ransomware detection
- Active response execution — runs local scripts when Manager triggers a response

---

### Wazuh Manager

| Field | Detail |
|---|---|
| **What it is** | Central brain of the Wazuh stack — receives, decodes, and analyses all agent data |
| **Role in stack** | Runs the rule engine, generates alerts, triggers active responses, and writes to the Indexer |
| **Port** | `1514` (agent comms), `1515` (agent registration), `55000` (REST API) |
| **Config location** | `docker-compose.yml` → Wazuh Manager service |
| **Custom rules** | `rules/` folder in this repo — mounted into the container |

The Manager's rule engine processes every incoming event against a library of decoders and rules. Custom rules in `rules/` extend the default set with project-specific detections. When a rule matches, the Manager generates a structured alert and writes it to the Wazuh Indexer.

---

### Wazuh Indexer (OpenSearch)

| Field | Detail |
|---|---|
| **What it is** | Distributed search and analytics engine — a hardened OpenSearch distribution maintained by Wazuh |
| **Role in stack** | Stores all raw alerts, logs, and AI-enriched alert data; serves as the data backend for the dashboard and AI microservice |
| **Port** | `9200` (REST API), `9300` (cluster comms) |
| **Indices used** | `wazuh-alerts-*` (raw), `wazuh-ai-enriched-*` (AI scored) |
| **Config location** | `docker-compose.yml` → Wazuh Indexer service |

The AI microservice reads from `wazuh-alerts-*` and writes enriched results to `wazuh-ai-enriched-*` — keeping raw and processed data separate and queryable independently.

---

### Wazuh Dashboard (OpenSearch Dashboards)

| Field | Detail |
|---|---|
| **What it is** | Web-based visualisation layer built on OpenSearch Dashboards |
| **Role in stack** | Primary SOC analyst interface — displays alert queues, heatmaps, geo maps, and KPI panels |
| **Port** | `5601` |
| **URL** | `http://localhost:5601` |
| **Config location** | `docker-compose.yml` → Wazuh Dashboard service |
| **Custom dashboards** | `dashboards/` folder — import via Saved Objects in OpenSearch |

Custom dashboard panels are exported as JSON and stored in `dashboards/` so they can be version-controlled and imported into any fresh deployment.

---

### Python AI Microservice

| Field | Detail |
|---|---|
| **What it is** | Custom FastAPI application — the AI enrichment layer |
| **Role in stack** | Polls OpenSearch for new alerts, runs anomaly scoring and UEBA, calls LLM API for summaries, writes enriched results back |
| **Port** | `8000` |
| **URL** | `http://localhost:8000/docs` (Swagger UI) |
| **Code location** | `ai/` folder |
| **Key files** | `main.py`, `anomaly.py`, `ueba.py`, `summarizer.py` |

**Internal modules:**

| Module | File | Purpose |
|---|---|---|
| Alert poller | `main.py` | FastAPI app — polls OpenSearch every 30s for new alerts |
| Anomaly scorer | `anomaly.py` | Isolation Forest model — scores each alert 0.00–1.00 |
| UEBA engine | `ueba.py` | Builds per-user login/behaviour baseline, flags deviations |
| LLM summariser | `summarizer.py` | Sends alert JSON to Claude/OpenAI API, returns plain-English summary |

---

### TheHive

| Field | Detail |
|---|---|
| **What it is** | Open-source Security Incident Response Platform (SIRP) |
| **Role in stack** | Receives auto-created incident cases from the AI microservice; used by L2 analysts for investigation and case management |
| **Port** | `9000` |
| **URL** | `http://localhost:9000` |
| **Config location** | `docker-compose.yml` → TheHive service |
| **Triggered by** | UC-012 — any alert with Wazuh severity level ≥ 12 |

---

### Shuffle SOAR

| Field | Detail |
|---|---|
| **What it is** | Open-source Security Orchestration, Automation and Response platform |
| **Role in stack** | Executes automated response workflows — IP blocking, Slack notifications, TheHive updates |
| **Port** | `3001` |
| **URL** | `http://localhost:3001` |
| **Config location** | `docker-compose.yml` → Shuffle service |
| **Triggered by** | UC-014 — brute force threshold met; UC-015 — malware process detected |

---

### VirusTotal Integration

| Field | Detail |
|---|---|
| **What it is** | External threat intelligence API |
| **Role in stack** | Auto-enriches alerts with file hash reputation and IP/domain intelligence |
| **Auth** | API key stored in `.env` as `VIRUSTOTAL_API_KEY` |
| **Called by** | `summarizer.py` in the AI microservice |
| **Triggered by** | UC-015 (file hash lookup), UC-022 (DNS domain lookup) |
| **Rate limit** | Free tier: 4 requests/min, 500/day — sufficient for PoC volume |

---

## Data Flow — Step by Step

The following traces the journey of a single suspicious event from endpoint to analyst screen:

```
Step 1 — EVENT OCCURS ON ENDPOINT
  A user fails SSH login 6 times in 45 seconds.
  Wazuh Agent captures each auth log entry in real time.

Step 2 — AGENT FORWARDS TO MANAGER
  Agent encrypts and sends log entries to Wazuh Manager on port 1514.
  Transmission happens within milliseconds of the event.

Step 3 — MANAGER DECODES THE LOG
  Manager runs the log entry through its decoder library.
  Decoder identifies it as an SSH authentication failure event.
  Structured fields extracted: source IP, username, timestamp, host.

Step 4 — RULE ENGINE CORRELATION
  Manager checks the decoded event against all active rules.
  Custom rule 100001 (brute_force.xml) matches:
    → 5+ failures from same IP within 60 seconds.
  Alert generated with severity level 12, rule ID 100001, MITRE T1110.

Step 5 — ALERT WRITTEN TO OPENSEARCH
  Alert JSON written to wazuh-alerts-* index in the Wazuh Indexer.
  Alert is now queryable and visible in raw form.

Step 6 — AI MICROSERVICE PICKS UP ALERT
  FastAPI poller detects new alert in wazuh-alerts-* (polls every 30s).
  Alert JSON passed through three enrichment modules in sequence:

    anomaly.py
    → Isolation Forest model scores the alert: 0.87 (high anomaly)
    → Score appended to alert object

    ueba.py
    → Checks source IP against user's known IP baseline
    → IP 45.33.22.11 not seen before for this user → flagged
    → UEBA flag appended to alert object

    summarizer.py
    → Alert JSON sent to LLM API (Claude / OpenAI)
    → Plain-English summary returned and appended
    → File hash submitted to VirusTotal if applicable

Step 7 — ENRICHED ALERT WRITTEN BACK
  Fully enriched alert written to wazuh-ai-enriched-* index.
  Contains: original alert + anomaly score + UEBA flag + LLM summary + VT result.

Step 8 — DASHBOARD UPDATES
  Real-time alert queue panel (UC-017) reads from wazuh-ai-enriched-*.
  New alert appears at top of queue (sorted by AI score descending).
  L1 analyst sees: rule description, affected host, AI score 0.87, LLM summary.

Step 9 — AUTOMATED RESPONSE (runs in parallel with Step 8)
  Severity ≥ 12 → TheHive API called → incident case auto-created (UC-012)
  Brute force rule fired → Shuffle workflow triggered → source IP blocked (UC-014)
  Slack notification sent to SOC channel.

Step 10 — ANALYST TRIAGE
  L1 analyst reads LLM summary, reviews AI score, checks UEBA flag.
  Decides to escalate → assigns TheHive case to L2 analyst.
  L2 investigates, correlates with other alerts, closes the incident.
```

---

## Network & Ports Reference

| Service | Port | Protocol | Connected by |
|---|---|---|---|
| Wazuh Manager — agent comms | `1514` | UDP/TCP | Wazuh Agents |
| Wazuh Manager — agent registration | `1515` | TCP | Wazuh Agents |
| Wazuh Manager — REST API | `55000` | HTTPS | AI microservice, Shuffle |
| Wazuh Indexer — REST API | `9200` | HTTPS | Wazuh Manager, AI microservice, Dashboard |
| Wazuh Indexer — cluster | `9300` | TCP | Internal (indexer cluster) |
| Wazuh Dashboard | `5601` | HTTPS | Browser (SOC analyst) |
| Python AI Microservice | `8000` | HTTP | Internal, browser (Swagger UI) |
| TheHive | `9000` | HTTP | AI microservice, Shuffle, browser |
| Shuffle SOAR | `3001` | HTTP | AI microservice, browser |
| VirusTotal API | `443` | HTTPS | AI microservice (outbound only) |
| Claude / OpenAI API | `443` | HTTPS | AI microservice (outbound only) |

> All inter-service communication happens over the internal Docker network (`siem-network`). Only ports `5601`, `9000`, and `3001` need to be accessible from the analyst's browser. All other ports are internal.

---

## Infrastructure Requirements

### Per-component memory estimate

| Component | Minimum RAM | Recommended RAM | Notes |
|---|---|---|---|
| Wazuh Manager | 2 GB | 4 GB | Rule engine, alert correlation |
| Wazuh Indexer | 4 GB | 8 GB | Largest consumer — needs JVM heap |
| Wazuh Dashboard | 512 MB | 1 GB | OpenSearch Dashboards UI |
| Python AI Microservice | 256 MB | 512 MB | FastAPI + scikit-learn model |
| TheHive | 1 GB | 2 GB | Runs on JVM |
| Shuffle SOAR | 512 MB | 1 GB | Workflow engine |
| OS overhead | 1 GB | 1.5 GB | Ubuntu 22.04 baseline |
| **Total** | **~9.5 GB** | **~18 GB** | |

### Recommended host specs

| Spec | Minimum | Recommended |
|---|---|---|
| **RAM** | 8 GB (Wazuh only) | 16 GB (full stack) |
| **CPU** | 2 cores | 4 cores |
| **Disk** | 60 GB free | 80 GB free |
| **OS** | Ubuntu 22.04 / WSL2 | Ubuntu 22.04 |
| **Docker** | 24.0+ | Latest |
| **Network** | 100 Mbps | 1 Gbps |

### Two-machine deployment (recommended for PoC)

This project is designed to run across two machines — one for development and one as the server hosting the stack:

| Machine | Role | Minimum spec |
|---|---|---|
| **Laptop A** | Development — write code, push to GitHub | Any machine with VS Code + Git |
| **Laptop B** | Server — runs full Docker stack | 16 GB RAM, 4 cores, 80 GB disk, Ubuntu 22.04 |

Laptop A connects to Laptop B via SSH or VS Code Remote SSH extension. Code is synced via GitHub — push from Laptop A, pull on Laptop B.

### Disk usage estimate (30-day operation, small environment)

| Data | Size |
|---|---|
| Wazuh Indexer — alert data | 20–50 GB |
| Docker images (all services) | ~8 GB |
| Wazuh Manager + rules | ~2 GB |
| TheHive case data | ~2 GB |
| AI models + Python environment | ~1 GB |
| **Total** | **~33–63 GB** |

---

*Next: [`03-use-cases.md`](03-use-cases.md) — full use case list with MITRE ATT&CK mapping.*
*Deployment steps: [`04-deployment.md`](04-deployment.md)*