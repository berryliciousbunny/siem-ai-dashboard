# 01 — Project Overview

> This document covers the problem, the solution, target users, project scope, and how this system fits into a real SOC workflow. For technical architecture see [`02-architecture.md`](02-architecture.md). For use cases see [`03-use-cases.md`](03-use-cases.md).

---

## Table of Contents

- [The Problem](#the-problem)
- [What This Project Is](#what-this-project-is)
- [Target Users](#target-users)
- [Scope](#scope)
- [SOC Workflow Integration](#soc-workflow-integration)
- [Why Open Source — Why Wazuh](#why-open-source--why-wazuh)

---

## The Problem

Modern Security Operations Centres face a compounding set of challenges that make effective threat detection and response increasingly difficult — even for well-resourced teams.

### 1. Alert Fatigue

The average SOC analyst receives hundreds to thousands of alerts per day. The overwhelming majority are false positives, duplicates, or low-priority noise that bury the signals that actually matter. Without intelligent prioritisation, L1 analysts spend most of their shift manually triaging alerts that go nowhere — leaving them burnt out and less attentive to the alerts that are genuine threats. Critical incidents get missed not because detection failed, but because they were buried under noise.

### 2. High Cost of Commercial SIEMs

Enterprise SIEM solutions like Splunk, IBM QRadar, and Microsoft Sentinel are powerful — but their licensing costs are prohibitive for smaller organisations, startups, academic institutions, and teams building internal tooling. Splunk alone can cost tens of thousands of dollars per year depending on data ingestion volume. This creates a significant gap: smaller teams that need SIEM capability the most often cannot afford the tools designed to provide it.

### 3. Slow, Manual Triage

Without AI assistance, every alert requires an analyst to manually look up the rule ID, cross-reference the affected host, check threat intelligence databases, and write up findings — before even deciding whether to escalate. Mean Time to Detect (MTTD) and Mean Time to Respond (MTTR) suffer as a result. Tier-1 response actions like blocking an offending IP or creating an incident ticket are done by hand, adding further delay during active incidents.

---

## What This Project Is

**siem-ai-dashboard** is an open-source, AI-enhanced Security Information and Event Management system built on top of Wazuh — one of the most widely deployed open-source SIEM platforms in the world. It extends Wazuh's native threat detection and log correlation capabilities with a custom Python AI layer that scores alerts for statistical anomalies, builds per-user behavioural profiles, and generates plain-English alert summaries using a Large Language Model (LLM) API. All of this is surfaced through a custom OpenSearch dashboard designed around the daily workflow of SOC analysts — from real-time alert triage through to automated incident response and compliance reporting. The entire stack runs on Docker Compose, costs $0 in licensing, and is designed to be deployable by a single engineer on commodity hardware.

---

## Target Users

### L1 SOC Analyst
The primary day-to-day user of the dashboard. L1 analysts handle initial alert triage — reviewing incoming alerts, making a first assessment of severity, and either resolving or escalating to L2. This system directly reduces their workload through AI severity scoring (so the most dangerous alerts surface first), LLM-generated plain-English summaries (so they don't need to manually decode every rule ID and log entry), and automated tier-1 responses like IP blocking and ticket creation that run without analyst intervention.

### L2 SOC Analyst
Handles escalated incidents, deeper forensic investigation, and threat hunting. L2 analysts benefit from the MITRE ATT&CK heatmap (to understand the tactics and techniques active in the environment), the incident timeline in TheHive (automatically populated from alert data), and the UEBA anomaly detection layer which surfaces subtle behavioural deviations that rule-based detection alone would miss.

### CISO / Security Manager
Uses the KPI dashboard and compliance reporting panels to track team performance and demonstrate security posture to leadership and auditors. Key metrics — MTTD, MTTR, alert volume trends, false positive rate, and compliance control status — are available at a glance without requiring the CISO to dig into raw alert data.

---

## Scope

### In Scope

| Area | Detail |
|---|---|
| **Wazuh deployment** | Full stack via Docker Compose — Manager, Indexer, Dashboard |
| **Log ingestion** | Endpoint agents (Windows + Linux), syslog, auth logs, Sysmon |
| **Custom detection rules** | 11 custom Wazuh XML rules mapped to MITRE ATT&CK |
| **AI anomaly detection** | Isolation Forest, UEBA per-user baseline, time-series spike detection |
| **LLM alert summarisation** | Plain-English summaries via Claude / OpenAI API |
| **Custom dashboard** | OpenSearch panels — alert queue, MITRE heatmap, geo map, KPI panel |
| **Integrations** | TheHive (incident management), Shuffle (SOAR), VirusTotal (threat intel) |
| **Active response** | Automated IP blocking, alert notifications via Slack/email |
| **Compliance reporting** | PCI-DSS, GDPR, HIPAA via Wazuh built-in compliance modules |
| **Documentation** | Full GitHub docs, deployment guide, architecture diagram, test results |

### Out of Scope

| Area | Reason |
|---|---|
| **Multi-tenant support** | PoC is single-organisation — multi-tenancy adds significant complexity |
| **Enterprise SSO / SAML** | Authentication integration deferred to production implementation |
| **Cloud-native deployment** | Designed for on-prem / self-hosted; cloud adaptation is a future phase |
| **Mobile application** | Dashboard is web-based only |
| **Custom SOAR playbooks beyond IP blocking** | Shuffle integration covers basic automation; full playbook library is future work |
| **Log sources beyond endpoints/network** | Cloud provider logs (AWS CloudTrail, Azure AD) are a planned future addition |

---

## SOC Workflow Integration

The following shows where this system sits in the analyst's daily workflow — from the moment an attack occurs to resolution:

```
1. ATTACK OCCURS
   └─ Adversary performs brute force, lateral movement,
      privilege escalation, or other malicious activity

2. DETECTION
   └─ Wazuh agent on the affected host captures the event
   └─ Wazuh Manager matches it against correlation rules
   └─ Alert is generated and written to OpenSearch Indexer

3. AI ENRICHMENT (Python microservice — runs automatically)
   ├─ Anomaly scorer assigns a confidence score (0.00–1.00)
   ├─ UEBA checks if behaviour deviates from user baseline
   └─ LLM generates a plain-English summary of the alert

4. DASHBOARD — L1 ANALYST VIEW
   └─ Alert appears in real-time queue with AI score + LLM summary
   └─ Analyst reads the summary and makes a triage decision in seconds
      ├─ Low score + known pattern → dismiss or monitor
      └─ High score + anomalous → escalate

5. AUTOMATED TIER-1 RESPONSE (runs in parallel, no analyst action needed)
   ├─ TheHive ticket auto-created with full alert context
   ├─ Source IP auto-blocked if brute force threshold met
   └─ File hash submitted to VirusTotal if malware suspected

6. ESCALATION → L2 ANALYST
   └─ L2 opens TheHive case — alert data, LLM summary, VT result already there
   └─ L2 investigates timeline, correlates with other alerts
   └─ Incident resolved and closed in TheHive

7. REPORTING
   └─ CISO views KPI panel — MTTD, MTTR, alert trends
   └─ Compliance panel shows PCI-DSS / GDPR control status
```

---

## Why Open Source — Why Wazuh

### The case for open source

Open-source security tooling has matured significantly over the past decade. The tools used in this stack — Wazuh, OpenSearch, TheHive, Shuffle — are not hobbyist alternatives. They are deployed in real corporate environments, government agencies, and MSSPs worldwide. Building on open source means full visibility into how the detection engine works, the ability to customise every component, no vendor lock-in, and zero licensing cost — freeing budget for the human expertise that actually makes a SOC effective.

### Why Wazuh specifically

Wazuh was chosen as the SIEM core for several reasons:

| Reason | Detail |
|---|---|
| **Active development** | Wazuh is actively maintained with frequent releases and a large community |
| **Feature breadth** | Built-in FIM, vulnerability detection, active response, compliance modules out of the box |
| **Log source coverage** | Supports Windows, Linux, macOS, cloud providers, firewalls, and custom log formats |
| **OpenSearch native** | Deep integration with OpenSearch makes custom dashboards and queries straightforward |
| **MITRE ATT&CK mapping** | Wazuh rules natively reference MITRE technique IDs — no manual mapping required |
| **Docker support** | Official Docker images make deployment reproducible and environment-agnostic |
| **Extensibility** | Custom XML rules, active response scripts, and API access make it easy to build on top of |

Wazuh provides a solid, production-tested detection foundation. This project's contribution is the AI enrichment layer, custom dashboard, and integrated SOC workflow built on top of it.

---

*Next: [`02-architecture.md`](02-architecture.md) — full stack diagram and component breakdown.*