# 03 — Use Cases

> **Total:** 24 use cases across 4 categories
> **Framework:** MITRE ATT&CK mapped where applicable
> **Detection engine:** Wazuh rules + Python AI microservice

---

## Table of Contents

- [🔴 Threat Detection](#-threat-detection)
- [🟡 AI Anomaly Detection](#-ai-anomaly-detection)
- [🟢 SOC Workflow & Automation](#-soc-workflow--automation)
- [🔵 Dashboard & Visibility](#-dashboard--visibility)
- [Coverage Summary](#coverage-summary)

---

## 🔴 Threat Detection

Rule-based detection handled by Wazuh's correlation engine. Each use case maps to a custom XML rule in the `rules/` folder.

---

### UC-001 — Brute Force Login Detection

| Field | Detail |
|---|---|
| **Data source** | Auth logs (`/var/log/auth.log`, Windows Security Event Log) |
| **MITRE technique** | T1110 — Brute Force |
| **Detection method** | Wazuh rule: 5+ failed login attempts from the same source IP within 60 seconds |
| **Severity** | High |
| **Response** | Alert fired → Shuffle triggers IP block via firewall → TheHive ticket auto-created |

**What it catches:** Automated credential stuffing attacks, password spraying, and manual brute force attempts against SSH, RDP, or web login portals.

**Example alert trigger:**
```
5 failed SSH login attempts from 192.168.1.105 in 45 seconds → rule 100001 fires
```

---

### UC-002 — Off-Hours Login Alert

| Field | Detail |
|---|---|
| **Data source** | Auth logs |
| **MITRE technique** | T1078 — Valid Accounts |
| **Detection method** | Wazuh rule: successful login between 7:00 PM – 8:00 AM or on weekends |
| **Severity** | Medium |
| **Response** | Alert fired → analyst reviews → escalate or dismiss |

**What it catches:** Compromised accounts being used outside business hours, insider threats working outside normal patterns, or attackers using stolen credentials.

**Example alert trigger:**
```
Successful login by user john.doe at 02:14 AM on Saturday → rule 100002 fires
```

---

### UC-003 — Privilege Escalation Detection

| Field | Detail |
|---|---|
| **Data source** | Windows Event Log (Event ID 4672, 4728), Syslog (`sudo` commands) |
| **MITRE technique** | T1068 — Exploitation for Privilege Escalation |
| **Detection method** | Wazuh rule: monitors `sudo` usage, UAC bypass events, new admin group membership |
| **Severity** | Critical |
| **Response** | Immediate alert → TheHive ticket → L2 analyst escalation |

**What it catches:** Attackers or malicious insiders attempting to gain admin/root access from a standard user account.

**Example alert trigger:**
```
User www-data runs sudo /bin/bash on production server → rule 100003 fires
```

---

### UC-004 — Lateral Movement Detection

| Field | Detail |
|---|---|
| **Data source** | Network logs, Windows Event Log (Event ID 4624 logon type 3) |
| **MITRE technique** | T1021 — Remote Services |
| **Detection method** | Wazuh rule: RDP or SMB connections originating from non-admin workstations to multiple internal hosts within a short window |
| **Severity** | Critical |
| **Response** | Alert fired → isolate source host via active response → TheHive ticket |

**What it catches:** Attackers moving through the network after initial compromise, attempting to reach high-value targets like domain controllers or file servers.

**Example alert trigger:**
```
Host WORKSTATION-04 initiates SMB connections to 6 internal hosts in 2 minutes → rule 100004 fires
```

---

### UC-005 — Ransomware File Activity (FIM)

| Field | Detail |
|---|---|
| **Data source** | Wazuh FIM (File Integrity Monitoring) |
| **MITRE technique** | T1486 — Data Encrypted for Impact |
| **Detection method** | Wazuh FIM rule: mass file rename or delete events (50+ files modified in under 30 seconds), especially with unknown extensions |
| **Severity** | Critical |
| **Response** | Immediate host isolation via active response → all analysts alerted → incident declared |

**What it catches:** Early-stage ransomware encryption activity before it spreads across the network.

**Example alert trigger:**
```
Process svchost.exe renames 80 files in C:\Users\Documents in 20 seconds → rule 100005 fires
```

---

### UC-006 — Data Exfiltration via Large Outbound Transfer

| Field | Detail |
|---|---|
| **Data source** | Network logs, firewall logs |
| **MITRE technique** | T1041 — Exfiltration Over C2 Channel |
| **Detection method** | Wazuh rule: outbound transfer exceeding 500 MB to a single external IP not in the whitelist |
| **Severity** | High |
| **Response** | Alert fired → VirusTotal lookup on destination IP → analyst review |

**What it catches:** Attackers or malicious insiders exfiltrating sensitive data to external servers or cloud storage.

**Example alert trigger:**
```
Host DB-SERVER sends 1.2 GB to external IP 45.33.22.11 over port 443 → rule 100006 fires
```

---

### UC-007 — Malicious Process Execution

| Field | Detail |
|---|---|
| **Data source** | EDR telemetry, Sysmon Event ID 1 (process creation) |
| **MITRE technique** | T1059 — Command and Scripting Interpreter |
| **Detection method** | Wazuh rule: known malicious process names, suspicious parent-child process chains (e.g. Word spawning PowerShell), execution from temp directories |
| **Severity** | High |
| **Response** | Alert fired → file hash sent to VirusTotal → TheHive ticket created |

**What it catches:** Malware execution, living-off-the-land attacks, and macro-based document exploits.

**Example alert trigger:**
```
WINWORD.EXE spawns powershell.exe with encoded command → rule 100007 fires
```

---

### UC-021 — PowerShell Obfuscation Detection

| Field | Detail |
|---|---|
| **Data source** | PowerShell Script Block Logging (Windows Event ID 4104) |
| **MITRE technique** | T1027 — Obfuscated Files or Information |
| **Detection method** | Wazuh rule: detects base64-encoded PowerShell commands, `Invoke-Expression`, `IEX`, `DownloadString` patterns in script block logs |
| **Severity** | High |
| **Response** | Alert fired → full script block logged → analyst reviews for payload |

**What it catches:** Attackers using encoded or obfuscated PowerShell to evade signature-based detection — one of the most common techniques in real-world intrusions.

**Example alert trigger:**
```
powershell.exe -enc SQBFAFgAIAAoAE4AZQB3AC0AT... → rule 100021 fires
```

---

### UC-022 — DNS Tunnelling Detection

| Field | Detail |
|---|---|
| **Data source** | DNS query logs |
| **MITRE technique** | T1071.004 — Application Layer Protocol: DNS |
| **Detection method** | Wazuh rule: abnormally long DNS query strings (>50 chars), high frequency of DNS queries to a single domain, queries for uncommon TLDs |
| **Severity** | High |
| **Response** | Alert fired → domain submitted to VirusTotal → analyst review |

**What it catches:** Attackers using DNS as a covert channel to exfiltrate data or maintain C2 communication — bypasses most firewalls since DNS is rarely blocked.

**Example alert trigger:**
```
Host queries aGVsbG8gd29ybGQ.evil-domain.com 300 times in 60 seconds → rule 100022 fires
```

---

### UC-023 — Failed MFA / 2FA Attempts

| Field | Detail |
|---|---|
| **Data source** | Auth logs, identity provider logs (Azure AD, Okta) |
| **MITRE technique** | T1110.001 — Password Guessing |
| **Detection method** | Wazuh rule: 3+ MFA failures for the same account within 5 minutes |
| **Severity** | Medium |
| **Response** | Alert fired → account temporarily flagged → analyst reviews for MFA fatigue attack |

**What it catches:** MFA fatigue attacks (spamming push notifications until the user accepts), SIM-swapping follow-ups, and compromised password + MFA bypass attempts.

**Example alert trigger:**
```
User sarah.lee fails MFA push 5 times in 3 minutes → rule 100023 fires
```

---

### UC-024 — USB / Removable Media Insertion

| Field | Detail |
|---|---|
| **Data source** | Windows Event Log (Event ID 6416), udev logs on Linux |
| **MITRE technique** | T1091 — Replication Through Removable Media |
| **Detection method** | Wazuh rule: new removable storage device connected to any monitored endpoint |
| **Severity** | Medium |
| **Response** | Alert fired → device ID and host logged → analyst reviews for policy violation or BadUSB |

**What it catches:** Insider threats copying data to USB drives, BadUSB attacks, and policy violations in restricted environments.

**Example alert trigger:**
```
USB mass storage device VID_0781 connected to WORKSTATION-12 at 11:34 PM → rule 100024 fires
```

---

## 🟡 AI Anomaly Detection

Handled by the Python AI microservice (`ai/`). These use cases go beyond static rules — the AI builds a baseline of normal behaviour and flags statistical deviations in real time.

---

### UC-008 — Unusual Login Time for User

| Field | Detail |
|---|---|
| **Data source** | Auth logs → OpenSearch |
| **AI method** | Isolation Forest + UEBA per-user time baseline |
| **Model input** | Login hour, day of week, user ID |
| **Severity** | Dynamic (based on anomaly score) |

**How it works:** The model learns each user's normal login window over 30 days. Any login outside 2 standard deviations of their personal baseline triggers an anomaly score. Different from UC-002 — this is personalised per user, not a fixed business-hours window.

**Example:** User normally logs in between 8–9 AM on weekdays. A login at 3 AM on a Tuesday scores 0.94 anomaly → flagged.

---

### UC-009 — Abnormal Data Volume Per User

| Field | Detail |
|---|---|
| **Data source** | Network logs, file access logs → OpenSearch |
| **AI method** | Statistical baseline (mean + std dev per user per day) |
| **Model input** | Bytes transferred, file count accessed, user ID, timestamp |
| **Severity** | Dynamic |

**How it works:** Tracks each user's daily data access and transfer volume. Flags when a user's activity exceeds 3 standard deviations above their 30-day average — a strong indicator of data staging or exfil prep.

**Example:** User averages 200 MB/day of file access. On Monday they access 4.5 GB → anomaly score 0.91 → flagged.

---

### UC-010 — New Device / IP for Known User

| Field | Detail |
|---|---|
| **Data source** | Auth logs → OpenSearch |
| **AI method** | Behavioral profiling — known device/IP whitelist per user |
| **Model input** | Source IP, device fingerprint, user ID |
| **Severity** | Medium–High |

**How it works:** Maintains a rolling 60-day profile of each user's known source IPs and device identifiers. Any login from a new, unseen source triggers an alert — especially high severity if combined with an off-hours flag (UC-002/UC-008).

**Example:** User has always logged in from 10.0.1.x range. Login arrives from 185.220.x.x (Tor exit node) → flagged as critical.

---

### UC-011 — Alert Spike Anomaly

| Field | Detail |
|---|---|
| **Data source** | Wazuh alerts → OpenSearch |
| **AI method** | Time-series anomaly detection (rolling z-score) |
| **Model input** | Alert count per 5-minute window, alert rule ID |
| **Severity** | High |

**How it works:** Monitors the volume of alerts over time. A sudden spike (e.g. 10x normal volume in a 5-minute window) likely indicates an active attack in progress, a misconfigured rule, or a log flood — all worth investigating immediately.

**Example:** Average 12 alerts per 5 minutes. Spike to 340 alerts in 5 minutes at 2:00 AM → anomaly flagged → L2 paged.

---

## 🟢 SOC Workflow & Automation

Integrations that reduce manual analyst effort and automate tier-1 response actions.

---

### UC-012 — Auto-Create Incident Ticket from Alert

| Field | Detail |
|---|---|
| **Tool** | TheHive |
| **Trigger** | Any alert with Wazuh severity level ≥ 12 |
| **Method** | Python microservice calls TheHive API on alert ingestion |

**What it does:** Automatically creates a structured incident case in TheHive with alert metadata (timestamp, source IP, affected host, rule ID, MITRE technique). Assigns to the on-call analyst. Eliminates manual ticket creation for every high-severity alert.

**Data written to ticket:**
- Alert ID and rule description
- Source and destination IP
- Affected hostname
- MITRE ATT&CK technique
- AI anomaly score (if applicable)
- LLM summary (UC-013)

---

### UC-013 — LLM Plain-English Alert Summary for L1 Analyst

| Field | Detail |
|---|---|
| **Tool** | Claude API / OpenAI API |
| **Trigger** | Every alert ingested by the AI microservice |
| **Method** | Alert JSON sent to LLM API → plain-English summary returned → stored in OpenSearch |

**What it does:** Converts raw technical alert data into a concise, human-readable summary. Helps L1 analysts who may not know every rule ID or MITRE technique to quickly understand what happened, why it matters, and what to do next.

**Example output:**
```
⚠️ Brute Force Detected
A large number of failed SSH login attempts were made against server PROD-DB-01
from external IP 45.33.22.11 between 02:10–02:11 AM. This pattern is consistent
with an automated credential stuffing attack. Recommend blocking the source IP
and reviewing successful logins from this host in the past 24 hours.
MITRE: T1110 | Severity: High | Anomaly score: 0.89
```

---

### UC-014 — Auto Block IP on Brute Force Threshold

| Field | Detail |
|---|---|
| **Tool** | Wazuh Active Response + Shuffle SOAR |
| **Trigger** | UC-001 fires (brute force rule) |
| **Method** | Wazuh active response script runs `iptables` / `firewall-cmd` block on source IP |

**What it does:** Automatically blocks the offending IP at the firewall level the moment the brute force rule fires — no analyst action required for tier-1 response. Shuffle logs the block action and adds it to the TheHive ticket.

**Block duration:** Configurable (default: 3600 seconds / 1 hour)

---

### UC-015 — File Hash Auto Lookup on Malware Alert

| Field | Detail |
|---|---|
| **Tool** | VirusTotal API |
| **Trigger** | UC-007 fires (malicious process execution) |
| **Method** | Python microservice extracts file hash from alert → queries VirusTotal API → appends result to alert |

**What it does:** When a suspicious process is detected, the file hash is automatically submitted to VirusTotal. The detection ratio (e.g. "47/72 engines detect this as malicious") is appended to the alert and the TheHive ticket — giving analysts immediate threat context without manual lookups.

---

### UC-016 — Compliance Report Generation

| Field | Detail |
|---|---|
| **Tool** | Wazuh built-in compliance modules |
| **Standards** | PCI-DSS, GDPR, HIPAA, NIST 800-53 |
| **Method** | Wazuh maps detected events to compliance requirements automatically |

**What it does:** Wazuh natively tags alerts with the relevant compliance framework requirements. The dashboard surfaces a compliance overview panel showing which controls are passing, failing, or need review — useful for audit preparation and CISO reporting.

---

## 🔵 Dashboard & Visibility

Custom panels built in OpenSearch Dashboards, exportable as JSON from `dashboards/`.

---

### UC-017 — Real-Time Alert Queue with AI Severity Score

**Panel type:** Data table (live, auto-refresh every 30s)

**Columns:**
- Timestamp
- Rule description
- Affected host
- Source IP
- MITRE technique
- Wazuh severity (1–15)
- AI anomaly score (0.00–1.00)
- LLM summary preview
- Status (New / Assigned / Closed)

**Purpose:** Primary working view for L1 analysts. AI score column allows quick triage — analysts sort by AI score descending to prioritise the highest-confidence anomalies first.

---

### UC-018 — MITRE ATT&CK Heatmap

**Panel type:** Heatmap / matrix visualisation

**What it shows:** A grid of all MITRE ATT&CK tactics (columns) and techniques (rows). Cells are coloured by alert count over the selected time range — darker = more detections. Empty cells = no coverage or no detections.

**Purpose:** Gives the SOC team an instant view of which attack techniques are being actively detected and which areas of the MITRE matrix have no coverage — useful for tuning and gap analysis.

---

### UC-019 — Geo Map of Attack Origins

**Panel type:** Coordinate map (OpenSearch Maps)

**What it shows:** World map with dots representing source IPs of inbound attack attempts. Dot size = alert volume. Colour = severity level.

**Purpose:** Instant situational awareness of where attacks are originating geographically. Useful for identifying targeted campaigns from specific regions and for compliance reporting.

---

### UC-020 — Analyst KPI Panel

**Panel type:** Metric tiles + time-series line charts

**Metrics tracked:**

| KPI | Definition |
|---|---|
| MTTD | Mean Time to Detect — avg time from event to alert firing |
| MTTR | Mean Time to Respond — avg time from alert to ticket closed |
| Alert volume | Total alerts per day/week/month |
| False positive rate | Alerts dismissed as FP / total alerts |
| AI score distribution | Histogram of anomaly scores across all alerts |
| Open vs closed tickets | TheHive case status breakdown |

**Purpose:** Gives SOC managers and the CISO a quick view of team efficiency and detection effectiveness over time.

---

## Coverage Summary

| Category | Count | Detection engine |
|---|---|---|
| 🔴 Threat detection | 11 | Wazuh XML rules |
| 🟡 AI anomaly detection | 4 | Python AI microservice |
| 🟢 SOC workflow & automation | 5 | TheHive, Shuffle, VirusTotal, LLM API |
| 🔵 Dashboard & visibility | 4 | OpenSearch Dashboards |
| **Total** | **24** | |

### MITRE ATT&CK Techniques Covered

| Technique ID | Name | Use Case |
|---|---|---|
| T1110 | Brute Force | UC-001, UC-023 |
| T1078 | Valid Accounts | UC-002 |
| T1068 | Exploitation for Privilege Escalation | UC-003 |
| T1021 | Remote Services | UC-004 |
| T1486 | Data Encrypted for Impact | UC-005 |
| T1041 | Exfiltration Over C2 Channel | UC-006 |
| T1059 | Command and Scripting Interpreter | UC-007 |
| T1027 | Obfuscated Files or Information | UC-021 |
| T1071.004 | Application Layer Protocol: DNS | UC-022 |
| T1091 | Replication Through Removable Media | UC-024 |

---

*For deployment and rule implementation details see [`docs/04-deployment.md`](04-deployment.md) and the `rules/` folder.*