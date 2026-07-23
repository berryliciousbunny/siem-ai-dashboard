# 06 — Roadmap

> **Current version:** v0.0.1 — Pre-alpha
> **Current phase:** Phase 1 — Documentation & PoC Planning 🚧
> **Last updated:** 2026

---

## Table of Contents

- [Phase Timeline](#phase-timeline)
- [Phase Detail](#phase-detail)
- [Milestone Markers](#milestone-markers)
- [Future Work — Post v1.0](#future-work--post-v10)
- [Tracking Progress](#tracking-progress)

---

## Phase Timeline

| Version | Phase | Duration | Status |
|---|---|---|---|
| `v0.0.1` | Phase 1 — Documentation & PoC Planning | Week 1–2 | 🚧 In progress |
| `v0.1.0` | Phase 2 — Wazuh Deployment | Week 3–4 | ⬜ Planned |
| `v0.2.0` | Phase 3 — Custom Detection Rules | Week 5–6 | ⬜ Planned |
| `v0.3.0` | Phase 4 — Custom OpenSearch Dashboard | Week 7–8 | ⬜ Planned |
| `v0.4.0` | Phase 5 — AI / ML Layer | Week 9–11 | ⬜ Planned |
| `v0.5.0` | Phase 6 — Integrations | Week 11–12 | ⬜ Planned |
| `v0.6.0` | Phase 7 — Testing & Red Team Simulation | Week 13 | ⬜ Planned |
| `v1.0.0` | Phase 8 — Finalise & Publish | Week 14 | ⬜ Planned |

---

## Phase Detail

---

### Phase 1 — Documentation & PoC Planning
**Version:** `v0.0.1` · **Duration:** Week 1–2 · **Status:** 🚧 In progress

**Goal:**
Establish the full project foundation before a single line of code is written. Document the problem, architecture, use cases, and deployment plan so the GitHub repo is immediately useful and credible from day one.

**Deliverables:**
- [x] `README.md` — project overview, goals, stack, quick start
- [x] `docs/01-overview.md` — problem statement, target users, scope
- [x] `docs/02-architecture.md` — system diagram, component breakdown, data flow
- [x] `docs/03-use-cases.md` — 24 use cases with MITRE ATT&CK mapping
- [ ] `docs/04-deployment.md` — step-by-step deployment guide
- [ ] `docs/05-ai-layer.md` — AI microservice design and model choices
- [x] `docs/06-roadmap.md` — this document
- [ ] GitHub repo folder structure pushed
- [ ] Architecture diagram created in Excalidraw

**Success criteria:**
Someone with no prior context can read the repo and fully understand what the project does, why it exists, how it works, and how to deploy it — without asking any questions.

---

### Phase 2 — Wazuh Deployment
**Version:** `v0.1.0` · **Duration:** Week 3–4 · **Status:** ⬜ Planned

**Goal:**
Get the full Wazuh stack running locally via Docker Compose with at least two monitored endpoints (one Windows, one Linux) sending live data into the dashboard.

**Deliverables:**
- [ ] `docker-compose.yml` — full Wazuh stack (Manager + Indexer + Dashboard)
- [ ] `.env.example` — environment variable template
- [ ] Wazuh agents installed and connected on test endpoints
- [ ] Log sources confirmed flowing into OpenSearch Indexer
- [ ] Default dashboards explored, gaps identified
- [ ] `docs/04-deployment.md` — complete deployment guide written

**Success criteria:**
`docker compose up -d` starts the full stack. At least one agent is connected and sending alerts. Wazuh Dashboard is accessible at `http://localhost:5601` and showing live data.

---

### Phase 3 — Custom Detection Rules
**Version:** `v0.2.0` · **Duration:** Week 5–6 · **Status:** ⬜ Planned

**Goal:**
Write and deploy all 11 custom Wazuh XML detection rules covering the threat detection use cases defined in `docs/03-use-cases.md`. Each rule is tested and mapped to a MITRE ATT&CK technique.

**Deliverables:**
- [ ] `rules/brute_force.xml` — UC-001, UC-023
- [ ] `rules/lateral_movement.xml` — UC-004
- [ ] `rules/ransomware_fim.xml` — UC-005
- [ ] `rules/privilege_escalation.xml` — UC-003
- [ ] `rules/off_hours_login.xml` — UC-002
- [ ] `rules/data_exfiltration.xml` — UC-006
- [ ] `rules/malicious_process.xml` — UC-007, UC-021
- [ ] `rules/dns_tunnelling.xml` — UC-022
- [ ] `rules/mfa_failures.xml` — UC-023
- [ ] `rules/usb_insertion.xml` — UC-024
- [ ] All rules tested by manually triggering each detection scenario

**Success criteria:**
All 11 rule files committed to `rules/`. Each rule fires correctly when the corresponding attack scenario is simulated. Zero false positives on baseline traffic.

---

### Phase 4 — Custom OpenSearch Dashboard
**Version:** `v0.3.0` · **Duration:** Week 7–8 · **Status:** ⬜ Planned

**Goal:**
Build a custom SOC analyst dashboard in OpenSearch Dashboards that surfaces the right data in the right format for L1 analysts and CISO-level reporting.

**Deliverables:**
- [ ] Real-time alert queue panel with AI score column (UC-017)
- [ ] MITRE ATT&CK heatmap visualisation (UC-018)
- [ ] Geo map of attack origin IPs (UC-019)
- [ ] Analyst KPI panel — MTTD, MTTR, alert volume (UC-020)
- [ ] L1 analyst view (detailed, alert-focused)
- [ ] CISO summary view (high-level, KPI-focused)
- [ ] `dashboards/soc-analyst-view.json` — exported dashboard JSON
- [ ] `dashboards/ciso-summary-view.json` — exported dashboard JSON

**Success criteria:**
Both dashboard views importable from JSON into a fresh OpenSearch instance. Alert queue auto-refreshes every 30 seconds. All four UC-017–020 panels rendering correctly with live data.

---

### Phase 5 — AI / ML Layer
**Version:** `v0.4.0` · **Duration:** Week 9–11 · **Status:** ⬜ Planned

**Goal:**
Build and deploy the Python AI microservice that enriches every alert with an anomaly score, UEBA behavioural flag, and plain-English LLM summary — and surfaces that enrichment in the dashboard.

**Deliverables:**
- [ ] `ai/main.py` — FastAPI app, OpenSearch poller
- [ ] `ai/anomaly.py` — Isolation Forest anomaly scorer
- [ ] `ai/ueba.py` — per-user behavioural baseline engine
- [ ] `ai/summarizer.py` — LLM alert summariser (Claude / OpenAI)
- [ ] `ai/requirements.txt` — Python dependencies
- [ ] `ai/Dockerfile` — containerised microservice
- [ ] AI score and LLM summary visible on alert queue dashboard panel
- [ ] `docs/05-ai-layer.md` — complete AI layer documentation written

**Success criteria:**
Every new alert in `wazuh-alerts-*` gets enriched within 60 seconds. Enriched alerts appear in `wazuh-ai-enriched-*` with anomaly score, UEBA flag, and LLM summary fields populated. Dashboard alert queue shows AI score column with values between 0.00–1.00.

---

### Phase 6 — Integrations
**Version:** `v0.5.0` · **Duration:** Week 11–12 · **Status:** ⬜ Planned

**Goal:**
Connect TheHive, Shuffle SOAR, and VirusTotal to complete the automated SOC response workflow. High-severity alerts should flow from detection to incident ticket to automated response without any analyst action.

**Deliverables:**
- [ ] TheHive deployed and connected — auto incident ticket creation (UC-012)
- [ ] Shuffle SOAR deployed — auto IP block workflow (UC-014)
- [ ] VirusTotal API integration — auto file hash enrichment (UC-015)
- [ ] Wazuh active response scripts configured
- [ ] Slack / email notifications configured for critical alerts
- [ ] End-to-end flow tested: alert fires → ticket created → IP blocked → Slack notified

**Success criteria:**
A simulated brute force attack results in: alert firing in Wazuh, AI enrichment applied, TheHive case created automatically, source IP blocked via active response, Slack notification sent — all within 2 minutes of the attack, with zero manual analyst steps.

---

### Phase 7 — Testing & Red Team Simulation
**Version:** `v0.6.0` · **Duration:** Week 13 · **Status:** ⬜ Planned

**Goal:**
Validate the full detection stack against simulated real-world attacks using Atomic Red Team. Tune rules to reduce false positives and document coverage gaps.

**Deliverables:**
- [ ] Atomic Red Team installed on test endpoint
- [ ] All 11 custom rules tested against corresponding ATT&CK techniques
- [ ] False positive rate measured and tuned
- [ ] MITRE ATT&CK coverage map documented
- [ ] `docs/07-test-results.md` — full test results written

**Success criteria:**
All 11 rules fire correctly against their simulated attack scenarios. False positive rate on baseline traffic is below 5%. MITRE coverage documented with detected vs undetected techniques clearly noted.

---

### Phase 8 — Finalise & Publish
**Version:** `v1.0.0` · **Duration:** Week 14 · **Status:** ⬜ Planned

**Goal:**
Polish the repo for public release. Record a demo, write the contributing guide, tag `v1.0.0`, and publish.

**Deliverables:**
- [ ] Demo screenshots added to README
- [ ] Demo video recorded (Loom / OBS) — alert → AI summary → TheHive ticket
- [ ] `CONTRIBUTING.md` written
- [ ] GitHub badges updated (license, last commit, version)
- [ ] All docs reviewed and cross-links verified
- [ ] `v1.0.0` release tagged on GitHub
- [ ] Published on LinkedIn and security communities

**Success criteria:**
A new engineer can clone the repo, follow `docs/04-deployment.md`, and have the full stack running with live alerts within 1 hour. The repo clearly communicates what it is, how to use it, and how to contribute.

---

## Milestone Markers

| Tag | Name | Description | Status |
|---|---|---|---|
| `v0.0.1` | Pre-alpha | Repo created, documentation in progress | 🚧 Now |
| `v0.1.0` | Docs complete | All docs written, repo structure pushed to GitHub | ⬜ |
| `v0.2.0` | Wazuh live | Full stack deployed, agents connected, data flowing | ⬜ |
| `v0.3.0` | Rules live | All 11 custom detection rules active and tested | ⬜ |
| `v0.4.0` | Dashboard live | Custom OpenSearch panels built and exported | ⬜ |
| `v0.5.0` | AI live | AI microservice enriching alerts in real time | ⬜ |
| `v0.6.0` | Integrated | TheHive + Shuffle + VirusTotal fully connected | ⬜ |
| `v0.7.0` | Tested | Red team simulation complete, rules tuned | ⬜ |
| `v1.0.0` | Published | Full stack polished, demo recorded, publicly announced | ⬜ |

---

## Future Work — Post v1.0

The following are intentionally out of scope for this PoC but represent natural next phases for production adoption:

### Detection & Intelligence
- [ ] **Sigma rule converter** — import community-written Sigma rules directly into Wazuh
- [ ] **MISP integration** — live threat intelligence feed from an internal MISP instance
- [ ] **Cloud log sources** — AWS CloudTrail, Azure AD, GCP audit logs
- [ ] **Container monitoring** — Docker and Kubernetes audit log ingestion

### AI / ML Improvements
- [ ] **ML model retraining pipeline** — automated weekly retraining on new alert data
- [ ] **False positive feedback loop** — analyst dismiss actions feed back into the model
- [ ] **Multi-model ensemble** — combine Isolation Forest + Autoencoder for better anomaly coverage
- [ ] **Local LLM option** — self-hosted LLM (Ollama + Mistral) for air-gapped environments

### Platform & Deployment
- [ ] **Cloud deployment** — Terraform scripts for AWS and Azure deployment
- [ ] **Multi-tenant support** — separate views and data isolation per organisation
- [ ] **Enterprise SSO / SAML** — integrate with Active Directory or Okta
- [ ] **High availability** — OpenSearch cluster with 3+ nodes for production resilience

### SOC Workflow
- [ ] **Custom SOAR playbooks** — expanded Shuffle automation beyond IP blocking
- [ ] **Analyst performance tracking** — per-analyst MTTD/MTTR metrics in KPI panel
- [ ] **Shift handover report** — auto-generated daily summary of active incidents
- [ ] **Mobile-responsive dashboard** — optimised view for on-call analysts on mobile

---

## Tracking Progress

Progress is tracked via GitHub Issues and the project board:

- 🐛 **Bug** — something broken in the stack
- ✨ **Enhancement** — improvement to existing functionality
- 📄 **Documentation** — doc additions or fixes
- 🔬 **Research** — spike or investigation task
- 🚀 **Release** — milestone / version tag

→ [Open Issues](../../issues)
→ [Project Board](../../projects)

Contributions are welcome at any phase — see [`CONTRIBUTING.md`](../CONTRIBUTING.md) for guidelines.

---

*For architecture details see [`02-architecture.md`](02-architecture.md).*
*For full use case list see [`03-use-cases.md`](03-use-cases.md).*