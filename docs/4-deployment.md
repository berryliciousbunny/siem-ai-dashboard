# 04 — Deployment Guide

> This document covers everything needed to get the full stack running from scratch — prerequisites, two-machine setup, configuration, service startup, agent connection, dashboard import, and troubleshooting. For architecture context see [`02-architecture.md`](02-architecture.md).

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Two-Machine Setup](#two-machine-setup)
- [Clone & Configure](#clone--configure)
- [Start the Stack](#start-the-stack)
- [Connect Wazuh Agents](#connect-wazuh-agents)
- [Import Custom Dashboards](#import-custom-dashboards)
- [Import Custom Detection Rules](#import-custom-detection-rules)
- [Start the AI Microservice](#start-the-ai-microservice)
- [Verify Everything is Working](#verify-everything-is-working)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Hardware (Laptop B — the server machine)

| Spec | Minimum | Recommended |
|---|---|---|
| RAM | 8 GB (Wazuh only) | 16 GB (full stack) |
| CPU | 2 cores | 4 cores |
| Disk | 60 GB free | 80 GB free |
| OS | Ubuntu 22.04 / WSL2 | Ubuntu 22.04 |
| Network | Same LAN as Laptop A | Same LAN as Laptop A |

### Software to install on Laptop B

**If running Ubuntu 22.04 natively:**
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Git
sudo apt install git -y

# Install Docker
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker

# Add your user to the docker group (so you don't need sudo every time)
sudo usermod -aG docker $USER
newgrp docker

# Install Docker Compose plugin
sudo apt install docker-compose-plugin -y

# Install Python 3.11
sudo apt install python3.11 python3.11-pip python3-pip -y

# Install OpenSSH Server (so Laptop A can connect)
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh

# Verify installs
docker --version
docker compose version
python3.11 --version
git --version
```

**If running Windows with WSL2 on Laptop B:**
```powershell
# Run in PowerShell as Administrator
wsl --install

# Restart Laptop B, then open Ubuntu from Start Menu
# Then follow the Ubuntu commands above inside WSL2
```

> ⚠️ **WSL2 note:** Docker Desktop for Windows must be installed and WSL2 integration enabled. Download from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop).

### Software to install on Laptop A (dev machine)

- [Git](https://git-scm.com)
- [VS Code](https://code.visualstudio.com)
- [Python 3.11+](https://python.org)

**VS Code extensions to install on Laptop A:**
```
Remote - SSH          (connect to Laptop B)
Python                (linting + autocomplete)
Docker                (manage containers visually)
GitLens               (better Git visibility)
REST Client           (test FastAPI endpoints)
XML (Red Hat)         (Wazuh rule syntax highlight)
Markdown All in One   (preview docs)
```

### Accounts needed

| Account | Where | Why |
|---|---|---|
| GitHub | [github.com](https://github.com) | Host the repo |
| VirusTotal | [virustotal.com](https://virustotal.com) | Free API key for hash lookups |
| Anthropic or OpenAI | [anthropic.com](https://anthropic.com) / [platform.openai.com](https://platform.openai.com) | LLM API key (or use Ollama for free) |
| Docker Hub | [hub.docker.com](https://hub.docker.com) | Pull Wazuh Docker images |

---

## Two-Machine Setup

### Step 1 — Find Laptop B's IP address

On Laptop B, run:
```bash
# Linux / WSL2
ip a | grep inet

# Look for something like 192.168.x.x or 10.0.x.x
# This is your local network IP — note it down
```

### Step 2 — Test SSH from Laptop A to Laptop B

On Laptop A terminal:
```bash
ssh your_username@LAPTOP_B_IP

# Example:
ssh vickii@192.168.1.105

# First connection will ask to confirm fingerprint — type yes
# Enter Laptop B password when prompted
```

If this works, you can now control Laptop B entirely from Laptop A's terminal.

### Step 3 — Set up VS Code Remote SSH

On Laptop A in VS Code:
1. Press `Ctrl+Shift+P` → type `Remote-SSH: Connect to Host`
2. Enter `your_username@LAPTOP_B_IP`
3. VS Code will open a new window connected to Laptop B
4. Open the terminal inside VS Code — it runs on Laptop B
5. All file editing and terminal commands now execute on Laptop B

### Step 4 — Git sync workflow

```
Laptop A (write code in VS Code Remote SSH)
    ↓  git add . && git commit -m "message" && git push
  GitHub (source of truth)
    ↓  git pull (on Laptop B when needed)
Laptop B (runs the Docker stack)
```

Since VS Code Remote SSH lets you edit files directly on Laptop B, you push from Laptop B to GitHub and pull on Laptop B — Laptop A is just the interface.

---

## Clone & Configure

### Step 1 — Clone the repo onto Laptop B

In your Laptop B terminal (via SSH or VS Code Remote):
```bash
cd ~
git clone https://github.com/YOUR_USERNAME/siem-ai-dashboard.git
cd siem-ai-dashboard
```

### Step 2 — Set vm.max_map_count (required for OpenSearch)

OpenSearch requires this kernel parameter or it will fail to start:
```bash
sudo sysctl -w vm.max_map_count=262144

# Make it permanent across reboots
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
```

### Step 3 — Create your .env file

```bash
cp .env.example .env
nano .env   # or: code .env (if in VS Code Remote)
```

Fill in every value — reference table below:

```env
# ── OpenSearch ──────────────────────────────────────
OPENSEARCH_HOST=localhost
OPENSEARCH_PORT=9200
OPENSEARCH_USER=admin
OPENSEARCH_PASSWORD=YourStrongPasswordHere   # change this

# ── Wazuh ───────────────────────────────────────────
WAZUH_API_URL=https://localhost:55000
WAZUH_API_USER=wazuh-wui
WAZUH_API_PASSWORD=YourStrongPasswordHere    # change this

# ── AI Microservice ──────────────────────────────────
POLL_INTERVAL=30                             # seconds between alert polls
ANOMALY_THRESHOLD=0.65                       # alerts above this score = high priority
UEBA_BASELINE_DAYS=60                        # days of history for user baseline

# ── LLM Provider (choose one) ───────────────────────
LLM_PROVIDER=anthropic                       # anthropic | openai | ollama | none
ANTHROPIC_API_KEY=your_anthropic_key_here
OPENAI_API_KEY=your_openai_key_here
OLLAMA_MODEL=mistral                         # only used if LLM_PROVIDER=ollama
OLLAMA_URL=http://localhost:11434            # only used if LLM_PROVIDER=ollama

# ── VirusTotal ───────────────────────────────────────
VIRUSTOTAL_API_KEY=your_virustotal_key_here
VIRUSTOTAL_ENABLED=true

# ── TheHive ──────────────────────────────────────────
THEHIVE_URL=http://localhost:9000
THEHIVE_API_KEY=your_thehive_key_here        # generated after first TheHive login

# ── Shuffle ──────────────────────────────────────────
SHUFFLE_URL=http://localhost:3001
SHUFFLE_API_KEY=your_shuffle_key_here        # generated after first Shuffle login
```

> ⚠️ **Security note:** Never commit your `.env` file to GitHub. It is already listed in `.gitignore`. Double-check with `git status` before every push.

---

## Start the Stack

### Step 1 — Start all services

```bash
# From the siem-ai-dashboard directory
docker compose up -d
```

This pulls all Docker images on first run (~8 GB total) and starts every service. First run takes 5–10 minutes depending on your internet speed.

### Step 2 — Watch startup logs

```bash
# Watch all services
docker compose logs -f

# Watch a specific service
docker compose logs -f wazuh.manager
docker compose logs -f wazuh.indexer
docker compose logs -f wazuh.dashboard
```

### Step 3 — Check all containers are running

```bash
docker compose ps
```

Expected output — all services should show `Up`:
```
NAME                    STATUS
wazuh.manager           Up
wazuh.indexer           Up
wazuh.dashboard         Up
thehive                 Up
shuffle                 Up
```

### Step 4 — Verify services are accessible

| Service | URL | Expected response |
|---|---|---|
| Wazuh Dashboard | https://localhost:5601 | Login page |
| Wazuh API | https://localhost:55000 | JSON response |
| TheHive | http://localhost:9000 | Login page |
| Shuffle | http://localhost:3001 | Login page |

> ⚠️ **Browser SSL warning:** Wazuh uses a self-signed certificate. Click "Advanced → Proceed" to continue past the browser warning. This is expected for a local deployment.

**Default credentials (change these immediately after first login):**

| Service | Username | Password |
|---|---|---|
| Wazuh Dashboard | `admin` | `SecretPassword` |
| TheHive | `admin@thehive.local` | `secret` |
| Shuffle | Set on first login | — |

### Step 5 — Change default passwords

```bash
# Change Wazuh admin password via API
curl -k -u admin:SecretPassword -X PUT \
  "https://localhost:55000/security/users/1" \
  -H "Content-Type: application/json" \
  -d '{"password": "YourNewStrongPassword"}'

# Update OPENSEARCH_PASSWORD in .env to match
# Then restart the stack
docker compose down && docker compose up -d
```

---

## Connect Wazuh Agents

Agents need to be installed on every endpoint you want to monitor. Replace `WAZUH_MANAGER_IP` with Laptop B's IP address in all commands below.

### Linux agent

```bash
# On the Linux endpoint to monitor
curl -so wazuh-agent.deb \
  https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.0-1_amd64.deb

sudo WAZUH_MANAGER=WAZUH_MANAGER_IP \
     WAZUH_AGENT_NAME=linux-endpoint-01 \
     dpkg -i ./wazuh-agent.deb

sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent

# Verify agent is running
sudo systemctl status wazuh-agent
```

### Windows agent

Run in PowerShell as Administrator on the Windows endpoint:
```powershell
# Download installer
Invoke-WebRequest -Uri "https://packages.wazuh.com/4.x/windows/wazuh-agent-4.7.0-1.msi" `
  -OutFile "wazuh-agent.msi"

# Install and connect to manager
msiexec.exe /i wazuh-agent.msi /q `
  WAZUH_MANAGER="WAZUH_MANAGER_IP" `
  WAZUH_AGENT_NAME="windows-endpoint-01"

# Start the agent service
NET START WazuhSvc
```

### Verify agents are connected

In your browser, go to `https://localhost:5601` → log in → navigate to:
**Wazuh → Agents**

You should see your connected agents listed with status **Active** and a green dot. If agents show as **Disconnected**, check the troubleshooting section below.

---

## Import Custom Dashboards

### Step 1 — Open OpenSearch Dashboard

Go to `https://localhost:5601` and log in.

### Step 2 — Navigate to Saved Objects

In the left sidebar:
`Stack Management → Saved Objects → Import`

### Step 3 — Import dashboard files

Click **Import** and upload each file from the `dashboards/` folder:

```
dashboards/soc-analyst-view.json     ← L1 analyst alert queue + panels
dashboards/ciso-summary-view.json    ← CISO KPI + compliance summary
```

Select **"Automatically overwrite conflicts"** if prompted.

### Step 4 — Open the dashboards

In the left sidebar:
`Dashboard → [select dashboard name]`

Both dashboards will initially show empty panels — they will populate as alerts flow in from connected agents.

---

## Import Custom Detection Rules

### Step 1 — Copy rules into the Wazuh Manager container

```bash
# From the siem-ai-dashboard directory on Laptop B
docker cp rules/brute_force.xml wazuh.manager:/var/ossec/etc/rules/brute_force.xml
docker cp rules/lateral_movement.xml wazuh.manager:/var/ossec/etc/rules/lateral_movement.xml
docker cp rules/ransomware_fim.xml wazuh.manager:/var/ossec/etc/rules/ransomware_fim.xml
docker cp rules/privilege_escalation.xml wazuh.manager:/var/ossec/etc/rules/privilege_escalation.xml
```

### Step 2 — Verify rule syntax

```bash
# Run inside the Manager container
docker exec wazuh.manager /var/ossec/bin/ossec-logtest -t
```

If rules have syntax errors, the output will show which file and line. Fix the XML and re-copy.

### Step 3 — Restart the Wazuh Manager to load new rules

```bash
docker compose restart wazuh.manager

# Wait ~30 seconds then verify Manager is back up
docker compose ps wazuh.manager
```

### Step 4 — Verify rules are loaded

```bash
# Check rules are listed in the Manager
docker exec wazuh.manager grep -r "100001" /var/ossec/etc/rules/
# Should return your brute_force.xml rule
```

---

## Start the AI Microservice

### Option A — Run via Docker Compose (recommended)

The AI microservice is included in `docker-compose.yml` and starts automatically with the rest of the stack:

```bash
docker compose up -d ai-service

# Check it's running
docker compose ps ai-service

# Watch its logs
docker compose logs -f ai-service
```

### Option B — Run directly with Python (for development)

```bash
cd ai

# Create a virtual environment
python3.11 -m venv venv
source venv/bin/activate          # Linux / Mac
# venv\Scripts\activate           # Windows

# Install dependencies
pip install -r requirements.txt

# Start the service
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

### Verify the AI microservice is running

Open `http://localhost:8000/docs` in your browser. You should see the FastAPI Swagger UI with all endpoints listed.

Test the health endpoint:
```bash
curl http://localhost:8000/health
# Expected: {"status": "ok", "model_loaded": true}
```

### If using Ollama (local LLM — free option)

```bash
# Install Ollama on Laptop B
curl -fsSL https://ollama.com/install.sh | sh

# Pull the Mistral model (~4 GB)
ollama pull mistral

# Verify Ollama is running
ollama list

# Update .env
LLM_PROVIDER=ollama
OLLAMA_MODEL=mistral
OLLAMA_URL=http://localhost:11434
```

---

## Verify Everything is Working

Run through this checklist after completing all setup steps:

### Infrastructure
- [ ] `docker compose ps` shows all services as `Up`
- [ ] Wazuh Dashboard accessible at `https://localhost:5601`
- [ ] TheHive accessible at `http://localhost:9000`
- [ ] Shuffle accessible at `http://localhost:3001`
- [ ] AI microservice health check returns `{"status": "ok"}` at `http://localhost:8000/health`

### Agents & Data Flow
- [ ] At least one agent shows **Active** in Wazuh → Agents
- [ ] Alerts are appearing in Wazuh Dashboard → Security Events
- [ ] OpenSearch index `wazuh-alerts-*` has documents:
  ```bash
  curl -k -u admin:YourPassword \
    "https://localhost:9200/wazuh-alerts-*/_count"
  # Expected: {"count": N} where N > 0
  ```

### AI Enrichment
- [ ] OpenSearch index `wazuh-ai-enriched-*` has documents (appears within 30–60 seconds of first alert):
  ```bash
  curl -k -u admin:YourPassword \
    "https://localhost:9200/wazuh-ai-enriched-*/_count"
  # Expected: {"count": N} where N > 0
  ```
- [ ] Enriched alerts contain `anomaly_score`, `ueba_flag`, and `llm_summary` fields:
  ```bash
  curl -k -u admin:YourPassword \
    "https://localhost:9200/wazuh-ai-enriched-*/_search?size=1" \
    | python3 -m json.tool | grep -E "anomaly_score|ueba_flag|llm_summary"
  ```

### Dashboard
- [ ] SOC analyst dashboard shows alert queue panel with data
- [ ] AI score column visible on alert queue
- [ ] MITRE ATT&CK heatmap rendering (may be sparse until more alerts arrive)

### Integrations
- [ ] Trigger a test brute force (5 failed SSH logins) → verify TheHive case auto-created
- [ ] Verify Slack/email notification received (if configured)

---

## Troubleshooting

---

### Docker runs out of memory

**Symptom:** Wazuh Indexer container exits immediately or `docker compose ps` shows it as `Exited`.

**Fix:**
```bash
# Check container exit reason
docker compose logs wazuh.indexer | tail -20

# If you see "max virtual memory areas" error:
sudo sysctl -w vm.max_map_count=262144

# If you see OOM (out of memory) errors — increase Docker memory limit
# On WSL2: edit C:\Users\YOUR_USER\.wslconfig
[wsl2]
memory=12GB

# Then restart WSL2
wsl --shutdown
# Reopen Ubuntu and restart the stack
docker compose up -d
```

---

### OpenSearch won't start

**Symptom:** Dashboard shows "Unable to connect to OpenSearch" or Indexer container keeps restarting.

**Fix:**
```bash
# Check indexer logs
docker compose logs wazuh.indexer

# Most common cause — vm.max_map_count not set
sudo sysctl -w vm.max_map_count=262144

# Second most common — wrong password in .env
# Verify OPENSEARCH_PASSWORD matches what's in docker-compose.yml
cat .env | grep OPENSEARCH_PASSWORD

# Reset and restart
docker compose down -v    # ⚠️ this deletes all stored data
docker compose up -d
```

---

### Wazuh agent not connecting

**Symptom:** Agent shows **Disconnected** or **Never connected** in the Wazuh Dashboard.

**Fix:**
```bash
# On the agent machine — check agent status
sudo systemctl status wazuh-agent

# Check agent logs for connection errors
sudo tail -f /var/ossec/logs/ossec.log

# Most common causes:
# 1. Wrong Manager IP in agent config
sudo nano /var/ossec/etc/ossec.conf
# Look for <address> tag — must match Laptop B's IP

# 2. Firewall blocking port 1514 on Laptop B
sudo ufw allow 1514/udp
sudo ufw allow 1514/tcp
sudo ufw allow 1515/tcp

# 3. Agent not registered — re-register
sudo /var/ossec/bin/agent-auth -m WAZUH_MANAGER_IP
sudo systemctl restart wazuh-agent
```

---

### AI microservice can't reach OpenSearch

**Symptom:** AI service logs show connection errors or `wazuh-ai-enriched-*` index stays empty.

**Fix:**
```bash
# Check AI service logs
docker compose logs ai-service

# Test OpenSearch connection from inside the AI container
docker exec ai-service curl -k \
  -u admin:YourPassword \
  "https://wazuh.indexer:9200/_cluster/health"

# Most common causes:
# 1. Wrong OPENSEARCH_PASSWORD in .env — update and restart
docker compose restart ai-service

# 2. OpenSearch not fully started yet — wait 60 seconds and retry
# 3. Container name mismatch — verify service name in docker-compose.yml
docker compose ps
```

---

### LLM API errors

**Symptom:** AI service logs show API authentication errors or `llm_summary` field is missing from enriched alerts.

**Fix:**
```bash
# Check your API key is set correctly
cat .env | grep API_KEY

# Test API key manually
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-sonnet-4-6","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}'

# If you want to avoid API costs entirely — switch to Ollama
# Update .env: LLM_PROVIDER=ollama
# Then follow the Ollama setup steps above
docker compose restart ai-service
```

---

### TheHive not receiving auto-tickets

**Symptom:** High-severity alerts fire in Wazuh but no cases appear in TheHive.

**Fix:**
```bash
# Check AI service logs for TheHive errors
docker compose logs ai-service | grep thehive

# Verify TheHive API key is correct
# In TheHive UI: Settings → API Keys → Create new key
# Copy key to .env THEHIVE_API_KEY

# Test TheHive connection manually
curl -H "Authorization: Bearer YOUR_THEHIVE_API_KEY" \
  http://localhost:9000/api/case

# Restart AI service after updating .env
docker compose restart ai-service
```

---

### Stopping and restarting the stack

```bash
# Stop all services (preserves data)
docker compose down

# Stop and delete all data (full reset)
docker compose down -v

# Restart a single service
docker compose restart wazuh.manager

# View resource usage
docker stats
```

---

*For architecture details see [`02-architecture.md`](02-architecture.md).*
*For AI layer configuration see [`05-ai-layer.md`](05-ai-layer.md).*
*For use cases and detection rules see [`03-use-cases.md`](03-use-cases.md).*