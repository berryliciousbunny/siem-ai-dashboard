# 01 — Project Overview

> This document covers the problem, the solution, target users, project scope, and how this system fits into a real SOC workflow. For technical architecture and design principles see [`02-architecture.md`](02-architecture.md). For use cases see [`03-use-cases.md`](03-use-cases.md).

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

Enterprise SIEM solutions like Splunk, IBM QRadar, and Microsoft Sentinel are powerful — but their licensing costs are prohibitive for smaller organisations, startups, academic institutions, and teams building internal tooling. This creates a significant gap: smaller teams that need SIEM capability the most often cannot afford the tools designed to provide it.

### 3. Slow, Manual Triage

Without assistance, every alert requires an analyst to manually look up the rule ID, cross-reference the affected host, check threat intelligence databases, and write up findings before even deciding whether to escalate. Mean Time to Detect (MTTD) and Mean Time to Respond (MTTR) suffer as a result. Tier-1 response actions like blocking an offending IP or creating an incident ticket are done by hand, adding further delay during active incidents.

### 4. AI Assistance in Security Tooling Carries Its Own Risks

Adding AI to a SIEM is not a purely additive improvement — it introduces a new set of concerns that a security-conscious project has to address rather than ignore: sensitive telemetry potentially leaving the organisation's network, a model's output being treated as ground truth when it is a probabilistic guess, and automated actions being triggered by something that cannot be fully audited step-by-step. This project treats the AI layer as something that needs its own boundaries, not just its own features.

---

## What This Project Is

**siem-ai-dashboard** is an open-source, AI-enhanced Security Information and Event Management system built on top of Wazuh — one of the most widely deployed open-source SIEM platforms in the world. It extends Wazuh's native threat detection and log correlation capabilities with a Python AI pipeline that scores alerts for statistical anomaly, builds per-user behavioural profiles, sanitizes each alert before it is ever turned into a prompt, and generates a plain-English explanation using a language model — self-hosted and running entirely on local infrastructure by default, with a cloud provider available as an explicit opt-in for anyone who wants it. All of this is surfaced through a custom OpenSearch dashboard designed around the daily workflow of SOC analysts, while automated responses — IP blocking, ticket creation — are driven by fixed policy rules rather than by the AI's own judgment. The entire stack runs on Docker Compose, costs $0 in licensing, and is designed to be deployable by a single engineer on commodity hardware.

---

## Target Users

### L1 SOC Analyst
The primary day-to-day user of the dashboard. L1 analysts handle initial alert triage — reviewing incoming alerts, making a first assessment, and either resolving or escalating. This system reduces their workload through the anomaly score (a second, independent prioritisation signal alongside Wazuh's own severity), a plain-English explanation for each alert (so they don't need to manually decode every rule ID), and automated tier-1 responses like IP blocking and ticket creation that run from fixed policy rules without requiring the analyst to act on the AI's explanation first.

### L2 SOC Analyst
Handles escalated incidents, deeper forensic investigation, and threat hunting. L2 analysts benefit from the MITRE ATT&CK heatmap, the incident timeline in TheHive (already populated with severity, anomaly score, UEBA context, and the AI's explanation), and the UEBA layer surfacing behavioural deviations that rule-based detection alone would miss.

### CISO / Security Manager
Uses the KPI dashboard and compliance reporting panels to track team performance and demonstrate security posture to leadership and auditors. Key metrics — MTTD, MTTR, alert volume trends, false positive rate, and compliance control status — are available at a glance.

---

## Scope

### In Scope

| Area | Detail |
|---|---|
| **Wazuh deployment** | Full stack via Docker Compose — Manager, Indexer, Dashboard |
| **Log ingestion** | Endpoint agents (Windows + Linux), syslog, auth logs, Sysmon |
| **Custom detection rules** | Custom Wazuh XML rules mapped to MITRE ATT&CK |
| **AI anomaly detection** | Isolation Forest, UEBA per-user baseline |
| **Data sanitization** | A dedicated stage stripping secrets/PII before any alert reaches a language model, local or cloud |
| **AI explanation** | Plain-English alert explanations via a local LLM by default, with a cloud provider available as an opt-in config change |
| **Custom dashboard** | OpenSearch panels — alert queue, MITRE heatmap, geo map, KPI panel — showing severity, anomaly score, UEBA, and threat intel as separate signals |
| **Integrations** | TheHive (incident management), Shuffle (SOAR), VirusTotal (threat intel) — all triggered by fixed policy rules, not by the AI's output |
| **Active response** | Automated IP blocking, alert notifications, with higher-impact actions routed through human approval |
| **Compliance reporting** | PCI-DSS, GDPR, HIPAA via Wazuh built-in compliance modules |
| **Documentation** | Full GitHub docs, deployment guide, architecture diagram, test results |

### Out of Scope

| Area | Reason |
|---|---|
| **Multi-tenant support** | PoC is single-organisation — multi-tenancy adds significant complexity |
| **Enterprise SSO / SAML** | Authentication integration deferred to production implementation |
| **Cloud-native deployment** | Designed for on-prem / self-hosted; cloud adaptation is a future phase |
| **Mobile application** | Dashboard is web-based only |
| **AI-driven automated response** | Deliberately out of scope by design, not just by phase — the AI layer explains, it does not act; see [Design Principles](02-architecture.md#design-principles) |
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
      (Wazuh Severity — signal #1)

3. AI ENRICHMENT (runs automatically, local by default)
   ├─ Anomaly scorer assigns an Anomaly Score (signal #2)
   ├─ UEBA checks if behaviour deviates from user baseline
   │  (signal #3)
   ├─ Alert is sanitized — secrets/PII stripped — before
   │  it is turned into a prompt
   └─ A local LLM generates a plain-English explanation;
      nothing about the alert leaves the network at this
      step unless a cloud provider was explicitly configured

4. DASHBOARD — L1 ANALYST VIEW
   └─ Alert appears in real-time queue with all signals shown
      separately: Wazuh Severity, Anomaly Score, UEBA, VT
      reputation, and the AI's explanation
   └─ Analyst reads the explanation as a recommendation and
      makes their own triage decision

5. POLICY-DRIVEN RESPONSE (runs in parallel, independent of
   what the AI explanation said)
   ├─ TheHive ticket auto-created if Wazuh severity crosses
      a fixed threshold
   ├─ Source IP auto-blocked if the brute-force rule's fixed
      threshold is met
   └─ File hash submitted to VirusTotal if a malware rule fired

6. ESCALATION → L2 ANALYST
   └─ L2 opens TheHive case — severity, anomaly score, UEBA
      context, VT result, and the AI explanation are already there
   └─ L2 investigates timeline, correlates with other alerts
   └─ Incident resolved and closed in TheHive

7. REPORTING
   └─ CISO views KPI panel — MTTD, MTTR, alert trends
   └─ Compliance panel shows PCI-DSS / GDPR control status
```

---

## Why Open Source — Why Wazuh

### The case for open source

Open-source security tooling has matured significantly over the past decade. The tools used in this stack — Wazuh, OpenSearch, TheHive, Shuffle — are not hobbyist alternatives; they are deployed in real corporate environments, government agencies, and MSSPs worldwide. Building on open source means full visibility into how the detection engine works, the ability to customise every component, no vendor lock-in, and zero licensing cost.

### Why Wazuh specifically

| Reason | Detail |
|---|---|
| **Active development** | Actively maintained with frequent releases and a large community |
| **Feature breadth** | Built-in FIM, vulnerability detection, active response, compliance modules out of the box |
| **Log source coverage** | Supports Windows, Linux, macOS, cloud providers, firewalls, and custom log formats |
| **OpenSearch native** | Deep integration with OpenSearch makes custom dashboards and queries straightforward |
| **MITRE ATT&CK mapping** | Wazuh rules natively reference MITRE technique IDs |
| **Docker support** | Official Docker images make deployment reproducible |
| **Extensibility** | Custom XML rules, active response scripts, and API access make it easy to build on top of |

Wazuh provides the detection foundation. This project's contribution is the AI enrichment layer — with its own sanitization boundary and local-by-default design — the custom dashboard, and the policy-driven response workflow built on top of it.

### Why local AI by default

The same reasoning that motivates open source infrastructure applies to the AI layer: an organisation's security telemetry is some of its most sensitive data, and a SOC tool that pipes raw alerts to a third-party API by default would be working against the same self-hosted, no-lock-in philosophy the rest of the stack is built on. Running the LLM locally via Ollama keeps that data inside the network, matches the project's zero-cost positioning, and is documented as the default rather than an afterthought — see [`02-architecture.md`](02-architecture.md#design-principles) and [`05-ai-layer.md`](05-ai-layer.md) for how that boundary is implemented.

---

*Next: [`02-architecture.md`](02-architecture.md) — full stack diagram, design principles, and component breakdown.*
