# 02 — Architecture

> This document covers the full system design, component breakdown, data flow, network reference, and infrastructure requirements. For project context see [`01-overview.md`](01-overview.md). For deployment steps see [`04-deployment.md`](04-deployment.md).

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Design Principles](#design-principles)
- [System Diagram](#system-diagram)
- [Component Breakdown](#component-breakdown)
- [Data Flow — Step by Step](#data-flow--step-by-step)
- [Network & Ports Reference](#network--ports-reference)
- [Infrastructure Requirements](#infrastructure-requirements)

---

## Architecture Overview

The system is built in five distinct layers. The **collection layer** gathers raw logs from monitored endpoints via Wazuh agents. The **detection layer** — Wazuh Manager and its rule engine — correlates incoming events against custom and built-in detection rules, generating structured alerts. The **storage layer** — the Wazuh Indexer — holds both raw and AI-enriched alerts as separate, queryable indices. The **AI enrichment layer** — a Python microservice — scores alerts for statistical anomaly, checks per-user behavioural baselines, sanitizes the alert before it ever reaches a language model, and produces a plain-English explanation via a **local** LLM by default. The **response layer** is kept deliberately separate from AI analysis: automated actions (IP blocking, ticket creation) are driven by explicit policy rules, not by the LLM directly, with an optional human-approval step for anything higher-impact than a routine block.

All components run as Docker containers orchestrated by `docker-compose.yml`, deployable on a single machine.

---

## Design Principles

These four principles shape every decision in this architecture, and are the reason it looks the way it does rather than a simpler "AI reads alert, AI acts" pipeline:

**1. SIEM telemetry stays inside the network by default.**
Alert data is never sent to a third-party API unless the operator explicitly opts into a cloud LLM provider. The default path runs entirely on local infrastructure.

**2. Nothing reaches the LLM unsanitized.**
Every alert passes through a dedicated sanitization step — stripping secrets, unnecessary PII, and normalizing fields — before it is turned into a prompt. This boundary exists regardless of which LLM provider is configured.

**3. The LLM explains; policy decides; a human or a fixed rule acts.**
The LLM's output is a natural-language explanation and a recommendation, not a command. Whether an action executes (blocking an IP, opening a ticket) is controlled by deterministic policy rules that sit between the AI layer and the response layer — never by the model's own judgment.

**4. Distinct signals stay distinct.**
Wazuh rule severity, the anomaly score, the UEBA deviation, and threat-intel reputation are four different kinds of evidence. They are surfaced separately on the dashboard rather than collapsed into a single blended number, so an analyst can see *why* something looks suspicious, not just *that* it does.

---

## System Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        MONITORED ENVIRONMENT                     │
│         Windows Hosts · Linux Servers · Network Devices          │
└───────────────────────────┬─────────────────────────────────────┘
                            │  Wazuh Agent (installed on each host)
                            │  Syslog / API log forwarders
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        WAZUH MANAGER                             │
│                                                                   │
│  Agents → Collection → Decoders → Rule Engine → Alerts           │
│                                                                   │
│  Custom rules: rules/brute_force.xml                             │
│                rules/lateral_movement.xml                        │
│                rules/ransomware_fim.xml                          │
│                rules/privilege_escalation.xml                    │
└────────────────────────────────────────────────────────┼─────────┘
                                                         │
                                                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                     WAZUH INDEXER (OpenSearch)                   │
│                                                                   │
│   wazuh-alerts-*          wazuh-ai-enriched-*                    │
│   (raw alerts)            (AI scored + LLM summary)              │
└──────────────┬─────────────────────────────────────┬─────────────┘
              │ raw alerts                          │ enriched alerts
              ▼                                      │
┌───────────────────────────────────────────────┐     │
│              AI ENRICHMENT PIPELINE            │     │
│                                                 │     │
│   Alert Retrieval  (main.py, polls every 30s)  │     │
│         ↓                                      │     │
│   Anomaly Detection  (anomaly.py)              │     │
│         ↓                                      │     │
│   UEBA  (ueba.py)                              │     │
│         ↓                                      │     │
│   Data Sanitization  (sanitizer.py)            │     │
│     — strip secrets / PII, normalize fields    │     │
│         ↓                                      │     │
│   LLM Gateway  (summarizer.py)                 │     │
│     — validates input, builds prompt,          │     │
│       routes to configured provider            │     │
│         ↓                                      │     │
│   ┌─────────────┬─────────────────┐            │     │
│   │  Local LLM  │  Cloud LLM      │            │     │
│   │  (default)  │  (opt-in)       │            │     │
│   │  Ollama     │  Claude/OpenAI  │            │     │
│   └─────────────┴─────────────────┘            │     │
│         ↓                                      │     │
│   Output Validation                            │     │
└──────────────────────┬──────────────────────────┘     │
                       │ writes enriched alert           │
                       └──────────────┬───────────────────┘
                                      ▼
                       ┌─────────────────────────────────┐
                       │       WAZUH DASHBOARD            │
                       │   (OpenSearch Dashboards :5601)  │
                       │                                   │
                       │  Alert Queue        (UC-017)      │
                       │  MITRE Heatmap      (UC-018)      │
                       │  Geo Attack Map     (UC-019)      │
                       │  Analyst KPI Panel  (UC-020)      │
                       │                                   │
                       │  Shows, per alert, as separate    │
                       │  signals — not one blended score: │
                       │   Severity · Anomaly · UEBA · VT  │
                       └────────────────┬──────────────────┘
                                       │
                                       ▼
                                 SOC ANALYST
                                       │
                       ┌───────────────┴────────────────┐
                       ▼                                 ▼
              Reviews AI explanation              Opens investigation
              as a recommendation                 in TheHive
                       │
                       ▼
        ┌─────────────────────────────────────────┐
        │            RESPONSE LAYER                 │
        │        (policy-driven, not AI-driven)      │
        │                                            │
        │  Wazuh Active Response  — fixed rule:       │
        │    brute force threshold → block IP         │
        │  Shuffle SOAR            — orchestrates      │
        │    the policy-approved workflow              │
        │  TheHive                 — case management   │
        │  VirusTotal              — reputation lookup  │
        │                                            │
        │  Higher-impact actions route through a      │
        │  human-approval step before executing.      │
        └─────────────────────────────────────────┘
```

---

## Component Breakdown

### Wazuh Agent

| Field | Detail |
|---|---|
| **What it is** | Lightweight endpoint agent installed on each monitored host |
| **Role in stack** | Collects logs, monitors file integrity, tracks running processes, forwards data to the Wazuh Manager |
| **Supported platforms** | Windows, Linux, macOS |
| **Communication** | Sends encrypted data to Wazuh Manager on port `1514` (UDP/TCP) |
| **Config location** | `/var/ossec/etc/ossec.conf` on each agent host |

Key capabilities used in this project: log collection (auth logs, Windows Event Log, Sysmon), File Integrity Monitoring for UC-005, and active response execution — the agent runs the local script when the Manager's policy triggers a response, but the decision to trigger stays outside the AI layer.

---

### Wazuh Manager

| Field | Detail |
|---|---|
| **What it is** | Central brain of the Wazuh stack — receives, decodes, and analyses all agent data |
| **Role in stack** | Runs the rule engine, generates alerts, executes policy-approved active responses, writes to the Indexer |
| **Port** | `1514` (agent comms), `1515` (agent registration), `55000` (REST API) |
| **Config location** | `docker-compose.yml` → Wazuh Manager service |
| **Custom rules** | `rules/` folder in this repo — mounted into the container |

The rule engine's output is the **Wazuh severity** signal — one of four distinct signals shown on the dashboard (see [Design Principles](#design-principles)). It is never overwritten or blended with the anomaly score.

---

### Wazuh Indexer (OpenSearch)

| Field | Detail |
|---|---|
| **What it is** | Distributed search and analytics engine — a hardened OpenSearch distribution maintained by Wazuh |
| **Role in stack** | Stores raw alerts and AI-enriched alerts as two separate indices; storage/search backend for the whole stack |
| **Port** | `9200` (REST API), `9300` (cluster comms) |
| **Indices used** | `wazuh-alerts-*` (raw), `wazuh-ai-enriched-*` (post-pipeline) |
| **Config location** | `docker-compose.yml` → Wazuh Indexer service |

The Indexer is storage, not a queue — the AI pipeline polls it for new documents rather than being pushed to. Keeping raw and enriched data in separate indices means the original alert is always recoverable even if a pipeline step is changed or re-run later.

---

### Wazuh Dashboard (OpenSearch Dashboards)

| Field | Detail |
|---|---|
| **What it is** | Web-based visualisation layer built on OpenSearch Dashboards |
| **Role in stack** | Primary SOC analyst interface |
| **Port** | `5601` |
| **URL** | `http://localhost:5601` |
| **Custom dashboards** | `dashboards/` folder — imported as Saved Objects |

Each alert on the dashboard shows Wazuh severity, anomaly score, UEBA deviation, and VirusTotal reputation as four labelled fields — never collapsed into a single number — so the analyst can see which kind of evidence is driving the alert.

---

### AI Enrichment Pipeline

| Field | Detail |
|---|---|
| **What it is** | A staged pipeline inside the Python microservice — not a single monolithic step |
| **Role in stack** | Turns a raw alert into an enriched one: anomaly score, behavioural context, a sanitized and explained version of the event |
| **Port** | `8000` (FastAPI, Swagger UI at `/docs`) |
| **Code location** | `ai/` folder |

**Pipeline stages, in order:**

| Stage | File | Purpose |
|---|---|---|
| Alert Retrieval | `main.py` | Polls `wazuh-alerts-*` every 30s for new documents |
| Anomaly Detection | `anomaly.py` | Isolation Forest — outputs an **Anomaly Score** (0.00–1.00), never called "severity" |
| UEBA | `ueba.py` | Per-user behavioural baseline — outputs a deviation flag and reason |
| Data Sanitization | `sanitizer.py` | Strips secrets and unnecessary PII, validates and normalizes fields — runs regardless of which LLM provider is configured |
| LLM Gateway | `summarizer.py` | Validates the sanitized input, builds the prompt, routes to the configured provider (local by default), validates the output before it's stored |

Anomaly detection and UEBA are statistical analysis; the LLM stage is semantic explanation. Keeping them as separate stages — rather than one "AI does everything" step — is what makes the sanitization boundary and the local-vs-cloud choice possible without touching the rest of the pipeline.

---

### LLM Gateway and Providers

| Field | Detail |
|---|---|
| **What it is** | The boundary between the sanitized alert and whichever model actually generates the explanation |
| **Default provider** | Local model via Ollama — alert data never leaves the machine/network |
| **Optional provider** | Claude API or OpenAI API — opt-in, configured via `.env` |
| **Why a gateway rather than a direct call** | Swapping providers, enforcing the sanitization boundary, and validating output are handled in one place instead of being duplicated per provider |

```env
LLM_PROVIDER=ollama          # default
# LLM_PROVIDER=anthropic     # opt-in upgrade path
# LLM_PROVIDER=openai        # opt-in upgrade path
```

The LLM's output is stored as an **explanation and recommendation** on the enriched alert. It is read-only from the response layer's perspective — nothing downstream executes an action because the LLM said to.

---

### TheHive

| Field | Detail |
|---|---|
| **What it is** | Open-source Security Incident Response Platform (SIRP) |
| **Role in stack** | Case management — receives auto-created incidents from the **policy layer**, not directly from the LLM |
| **Port** | `9000` |
| **Triggered by** | Policy rule: Wazuh severity ≥ 12 (a fixed threshold, independent of the AI explanation) |

---

### Shuffle SOAR

| Field | Detail |
|---|---|
| **What it is** | Open-source Security Orchestration, Automation and Response platform |
| **Role in stack** | Executes policy-approved response workflows — never triggered by the LLM's output directly |
| **Port** | `3001` |
| **Triggered by** | Fixed policy rule (e.g. brute force threshold met), with higher-impact actions routed through human approval before executing |

---

### VirusTotal Integration

| Field | Detail |
|---|---|
| **What it is** | External threat intelligence API |
| **Role in stack** | Reputation lookup — one of the four distinct signals shown on the dashboard, alongside severity, anomaly score, and UEBA |
| **Auth** | API key in `.env` as `VIRUSTOTAL_API_KEY` |
| **Rate limit** | Free tier: 4 requests/min, 500/day |

---

## Data Flow — Step by Step

Tracing a single suspicious event from endpoint to response:

```
Step 1 — EVENT OCCURS ON ENDPOINT
  A user fails SSH login 6 times in 45 seconds.
  Wazuh Agent captures each auth log entry in real time.

Step 2 — AGENT FORWARDS TO MANAGER
  Encrypted transmission to Wazuh Manager on port 1514.

Step 3 — MANAGER DECODES AND CORRELATES
  Decoder extracts structured fields: source IP, username, timestamp, host.
  Rule engine matches custom rule 100001 (brute_force.xml):
    5+ failures from same IP within 60 seconds.
  Alert generated: Wazuh severity level 12, rule ID 100001, MITRE T1110.

Step 4 — ALERT WRITTEN TO OPENSEARCH
  Written to wazuh-alerts-* — this is the raw signal, untouched.

Step 5 — AI PIPELINE PICKS UP THE ALERT
  main.py polls and retrieves the new alert.

  anomaly.py
  → Isolation Forest scores it: 0.87 Anomaly Score (not "severity")

  ueba.py
  → Source IP not seen before for this user → UEBA deviation: high

  sanitizer.py
  → Strips anything unnecessary (raw payloads, internal-only fields)
  → Validates and normalizes what remains before it can reach an LLM

  summarizer.py (LLM Gateway)
  → Sanitized alert sent to the configured provider — local model by
    default, so nothing leaves the network at this step
  → Plain-English explanation returned and validated
  → File hash / IP submitted to VirusTotal if applicable

Step 6 — ENRICHED ALERT WRITTEN BACK
  wazuh-ai-enriched-* now holds: original alert + Anomaly Score +
  UEBA deviation + VT reputation + LLM explanation — four distinct
  fields, not one blended number.

Step 7 — DASHBOARD UPDATES
  Alert queue (UC-017) shows all four signals side by side.
  L1 analyst reads the LLM's explanation as a recommendation, not
  an instruction, and makes their own triage call.

Step 8 — POLICY LAYER EVALUATES — INDEPENDENTLY OF THE LLM
  Wazuh severity ≥ 12 → policy rule → TheHive case auto-created.
  Brute force rule fired → policy rule → Shuffle blocks the source IP.
  Neither action was decided by the LLM; both fire from the fixed
  rule regardless of what the explanation says.

Step 9 — ANALYST TRIAGE
  L1 analyst reviews the case in TheHive, already populated with
  severity, anomaly score, UEBA context, VT result, and the LLM's
  explanation. Escalates to L2 if warranted.
```

---

## Network & Ports Reference

| Service | Port | Protocol | Connected by |
|---|---|---|---|
| Wazuh Manager — agent comms | `1514` | UDP/TCP | Wazuh Agents |
| Wazuh Manager — agent registration | `1515` | TCP | Wazuh Agents |
| Wazuh Manager — REST API | `55000` | HTTPS | AI pipeline, Shuffle |
| Wazuh Indexer — REST API | `9200` | HTTPS | Wazuh Manager, AI pipeline, Dashboard |
| Wazuh Dashboard | `5601` | HTTPS | Browser (SOC analyst) |
| Python AI Microservice | `8000` | HTTP | Internal, browser (Swagger UI) |
| Ollama (local LLM) | `11434` | HTTP | AI pipeline only — never exposed externally |
| TheHive | `9000` | HTTP | AI pipeline (policy layer), Shuffle, browser |
| Shuffle SOAR | `3001` | HTTP | AI pipeline (policy layer), browser |
| VirusTotal API | `443` | HTTPS | AI pipeline (outbound only) |
| Cloud LLM API (opt-in) | `443` | HTTPS | AI pipeline (outbound only, only if configured) |

> Only `5601`, `9000`, and `3001` need to be reachable from the analyst's browser. Everything else stays on the internal Docker network. Ollama in particular is never exposed outside the container network — it has no reason to be reachable from anywhere but the AI pipeline.

---

## Infrastructure Requirements

Running a local LLM changes the RAM picture significantly, so this project documents two deployment profiles instead of one blended estimate.

### Core SIEM profile (no AI layer running)

| Component | Minimum RAM | Recommended RAM |
|---|---|---|
| Wazuh Manager | 2 GB | 4 GB |
| Wazuh Indexer | 4 GB | 8 GB |
| Wazuh Dashboard | 512 MB | 1 GB |
| OS overhead | 1 GB | 1.5 GB |
| **Total** | **~7.5 GB** | **~14.5 GB** |

This profile is enough to complete Phase 2 and Phase 3 (deployment, custom rules) without the AI layer running yet.

### Full stack profile (AI layer + local LLM)

| Component | Minimum RAM | Recommended RAM |
|---|---|---|
| Core SIEM (above) | 7.5 GB | 14.5 GB |
| Python AI Microservice | 256 MB | 512 MB |
| Local LLM (Ollama, 7B model) | 4 GB | 8 GB |
| TheHive | 1 GB | 2 GB |
| Shuffle SOAR | 512 MB | 1 GB |
| **Total** | **~13.5 GB** | **~26.5 GB** |

> On a 16 GB machine, running the full profile with a local 7B model at the same time as Core SIEM is tight — expect to close other applications, and consider running the local LLM and TheHive/Shuffle at different times rather than all simultaneously during early testing. A cloud LLM provider removes the 4–8 GB local-model line if hardware is the binding constraint, at the cost of sending sanitized (not raw) alert data externally.

### Recommended host specs

| Spec | Minimum (Core SIEM) | Recommended (Full stack) |
|---|---|---|
| **RAM** | 8 GB | 16 GB, ideally more with a local LLM in play |
| **CPU** | 2 cores | 4 cores |
| **Disk** | 60 GB free | 80 GB free (add ~4–5 GB for a local model) |
| **OS** | Ubuntu 22.04 / WSL2 | Ubuntu 22.04 |
| **Docker** | 24.0+ | Latest |

### Two-machine deployment

| Machine | Role | Minimum spec |
|---|---|---|
| **Laptop A** | Development — write code, push to GitHub | Any machine with VS Code + Git |
| **Laptop B** | Server — runs the Docker stack | 16 GB RAM, 4 cores, 80 GB disk, Ubuntu 22.04 / WSL2 |

### Disk usage estimate (30-day operation, small environment)

| Data | Size |
|---|---|
| Wazuh Indexer — alert data | 20–50 GB |
| Docker images (all services) | ~8 GB |
| Local LLM model file | ~4–5 GB |
| Wazuh Manager + rules | ~2 GB |
| TheHive case data | ~2 GB |
| **Total** | **~36–67 GB** |

---

*Next: [`03-use-cases.md`](03-use-cases.md) — full use case list with MITRE ATT&CK mapping.*
*AI layer detail: [`05-ai-layer.md`](05-ai-layer.md)*
