<div align="center">

# SOC Automation Homelab

<p>
  <img src="https://img.shields.io/badge/Cloud-Microsoft%20Azure-0078D4?style=flat-square&logo=microsoftazure&logoColor=white" alt="Azure">
  <img src="https://img.shields.io/badge/SIEM-Wazuh%204.9-00ADEF?style=flat-square" alt="Wazuh">
  <img src="https://img.shields.io/badge/SOAR-Shuffle%20(self--hosted)-6B7CF6?style=flat-square" alt="Shuffle">
  <img src="https://img.shields.io/badge/Case_Management-TheHive%205-FF6B35?style=flat-square" alt="TheHive">
  <img src="https://img.shields.io/badge/Threat_Intel-VirusTotal-394EFF?style=flat-square" alt="VirusTotal">
  <img src="https://img.shields.io/badge/Status-Active-success?style=flat-square" alt="Status">
</p>

**A fully automated Security Operations Center (SOC) built end-to-end on Microsoft Azure using open-source detection, automation, and case management tools.**

*Demonstrates real threat detection, SOAR automation, threat intelligence enrichment, and incident case management — the same workflow used by enterprise SOC teams.*

</div>

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tools Used](#tools-used)
- [Infrastructure](#infrastructure)
- [How It Works](#how-it-works)
- [Test Case — Mimikatz Detection](#test-case--mimikatz-detection)
- [Engineering Challenges & Solutions](#engineering-challenges--solutions)
- [Screenshots](#screenshots)
- [Skills Demonstrated](#skills-demonstrated)
- [How to Replicate](#how-to-replicate)
- [License](#license)

---

## Overview

This project builds a fully functional mini-SOC that detects threats on a Windows endpoint in real time, enriches alerts with threat intelligence from 70+ antivirus engines, and automatically opens structured investigation cases — with zero manual intervention.

**What makes this different from just reading about SOC tools:**
- Every component is actually deployed and running on real cloud infrastructure, not simulated
- The full pipeline was validated against real attack tooling (Mimikatz, credential dumping via LSASS access)
- The architecture mirrors how production SOC environments are built — SIEM, SOAR, threat intel, and case management as separate, integrated systems
- Built and debugged independently, including resolving real infrastructure failures along the way (see [Engineering Challenges](#engineering-challenges--solutions))

---

## Architecture

```
┌──────────────────┐   Sysmon events    ┌──────────────────────┐
│  Windows 10 VM    │ ─────────────────► │   Wazuh SIEM          │
│  (VirtualBox,     │                    │   Azure VM: wazuh-vm  │
│   local PC)        │                    │   Ubuntu 24.04        │
│                    │                    │                       │
│  • Sysmon          │                    │  • Log ingestion      │
│  • Wazuh Agent     │                    │  • Rule correlation   │
└──────────────────┘                    │  • Alert firing       │
                                         └──────────┬────────────┘
                                                    │ Webhook (JSON)
                                                    ▼
                                         ┌───────────────────────┐
                                         │   Shuffle SOAR         │
                                         │   Self-hosted (Docker) │
                                         │   Azure VM: thehive-vm │
                                         │                        │
                                         │  • Receives alert      │
                                         │  • Regex-extracts hash │
                                         └──────────┬────────────┘
                                                    │ hash lookup
                                                    ▼
                                         ┌───────────────────────┐
                                         │   VirusTotal API v3    │
                                         │                        │
                                         │  • 70+ AV engines      │
                                         │  • Detection ratio     │
                                         └──────────┬────────────┘
                                                    │ enriched data
                                                    ▼
                                         ┌───────────────────────┐
                                         │   TheHive               │
                                         │   Self-hosted (Docker) │
                                         │   Azure VM: thehive-vm │
                                         │                        │
                                         │  • Creates alert/case  │
                                         └───────────────────────┘
```

---

## Tools Used

| Tool | Role | Cost |
|------|------|------|
| [Wazuh](https://wazuh.com) | SIEM / XDR — threat detection and log analysis | Free (open-source) |
| [Shuffle](https://shuffler.io) | SOAR — security orchestration, self-hosted via Docker | Free (open-source) |
| [TheHive](https://thehive-project.org) | Case management — incident investigation, self-hosted via Docker | Free (open-source) |
| [VirusTotal](https://virustotal.com) | Threat intelligence — IOC/hash enrichment | Free tier (API v3) |
| [Sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) | Windows telemetry — advanced event logging | Free (Microsoft) |
| [Docker](https://docker.com) | Container runtime for Shuffle + TheHive | Free |
| [Microsoft Azure](https://azure.microsoft.com) | Cloud hosting | Free trial credit |
| [VirtualBox](https://virtualbox.org) | Local hypervisor for Windows endpoint | Free |

---

## Infrastructure

| Component | Specs | Hosting |
|-----------|-------|---------|
| Wazuh Server | 2 vCPU · 8 GB RAM · Ubuntu 24.04 LTS | Azure VM `wazuh-vm` |
| Shuffle + TheHive Server | 2 vCPU · 8 GB RAM · Ubuntu 24.04 LTS · Docker | Azure VM `thehive-vm` |
| Windows Client | 4 GB RAM · Windows 10 Pro | VirtualBox (local) |

Both Azure VMs run in the same Virtual Network with Network Security Group rules scoped to only the required ports (22, 443, 1514, 1515, 55000, 9000, 3001).

---

## How It Works

**Step 1 — Telemetry Collection**
Sysmon runs on the Windows VM and captures detailed event data — process creation, image loads, network connections, and process access events (e.g. a process reading LSASS memory). The Wazuh Agent forwards these events to the Wazuh Manager in real time over an encrypted channel.

**Step 2 — Threat Detection**
Wazuh evaluates incoming events against its built-in ruleset, mapped to MITRE ATT&CK techniques. When a high-confidence rule matches — for example, a process accessing `lsass.exe` with read permissions, a known credential-dumping pattern (T1003) — Wazuh fires a severity-rated alert.

**Step 3 — SOAR Trigger**
Wazuh's integration module forwards the alert as JSON to a Shuffle webhook. Shuffle is self-hosted via Docker on the same VM as TheHive, triggering the automation workflow within seconds of the alert firing.

**Step 4 — Hash Extraction**
Sysmon's raw hash field returns multiple hash types in one string (`SHA1=...,MD5=...,SHA256=...`). A Shuffle Tools regex-capture step parses this and extracts a clean SHA256 hash for use in the next step.

**Step 5 — Threat Intelligence Enrichment**
The extracted hash is submitted to the VirusTotal API. VirusTotal returns the file's detection ratio across 70+ antivirus engines (e.g. 61 of 67 engines flagging it malicious), confirming whether the detected binary is known malware.

**Step 6 — Case Creation**
Shuffle sends a structured HTTP POST to TheHive's REST API, creating an alert that includes the Wazuh rule description, the affected agent, the rule ID, and the VirusTotal detection summary — all generated automatically, with no analyst input required.

**Step 7 — Analyst Review**
The SOC analyst opens TheHive and finds a fully enriched, ready-to-triage case waiting for them.

---

## Test Case — Mimikatz Detection

Mimikatz is a widely recognized credential-dumping tool and the industry-standard way to validate SOC detection pipelines.

**Test environment:** Isolated VirtualBox VM, no access to real credentials or production data.

| Step | Action | Result |
|------|--------|--------|
| 1 | Ran `mimikatz.exe` with `sekurlsa::logonpasswords` in the isolated Windows VM | Sysmon logged process access to `lsass.exe` (Event ID 10) |
| 2 | Wazuh evaluated the event | High-severity alert fired — rule 92900, MITRE T1003.001 (LSASS Memory) |
| 3 | Alert forwarded to self-hosted Shuffle via webhook | Workflow triggered within seconds |
| 4 | Shuffle extracted the SHA256 hash via regex | Clean hash isolated from the raw multi-hash Sysmon field |
| 5 | Shuffle queried VirusTotal | 61 of 67 engines flagged the file as malicious |
| 6 | Shuffle posted the enriched alert to TheHive | Case created with full Wazuh + VirusTotal context |

**Total pipeline time: under 60 seconds, fully automated.**

---

## Engineering Challenges & Solutions

Real infrastructure breaks in ways tutorials don't cover. Documenting how these were diagnosed and fixed:

**1. Wazuh manager crash — disk filled to 100%**
The `wazuh-manager` service failed silently after editing its config. Root-caused via `journalctl` to a `/var/ossec/queue/vd_updater/` cache file that had grown to several GB, filling the VM's entire disk. Resolved by clearing the cache, disabling the vulnerability-detector module (not needed for this use case), and adding a scheduled cleanup job to prevent recurrence.

**2. SSH connection timeouts**
Direct SSH from a local machine to the Azure VM consistently timed out, despite correct NSG and `ufw` rules. Diagnosed as the local ISP blocking outbound port 22. Resolved by using Azure Cloud Shell, which connects to the VM over Azure's internal network rather than the public internet.

**3. Hash field parsing for VirusTotal**
Sysmon's `hashes` field returns a combined string (`SHA1=...,MD5=...,SHA256=...`) which VirusTotal's API rejects outright. Solved by adding a regex-capture step in the Shuffle workflow to isolate a clean SHA256 value before the API call — and by identifying which specific Sysmon event types (Event ID 7 — Image Loaded) actually populate the hash field, since not all event types do.

**4. TheHive 403 authorization errors**
API calls to create alerts returned `403: missing permission manageAlert`. Traced to the default `admin@thehive.local` account being a platform-level admin with no organization membership — alerts require an org-scoped user. Resolved by creating a dedicated organization and an `org-admin` profile user, then re-issuing the API key.

**5. Shuffle free-tier execution limit reached**
After exhausting the 2,000 free monthly cloud executions, migrated Shuffle to a self-hosted Docker deployment on the existing Azure VM — removing the execution cap entirely and adding genuine infrastructure depth to the project.

---

## Screenshots

### 1. Wazuh — Mimikatz credential-dumping alert
![Wazuh Alert](screenshots/Event-Logs.png)
*Wazuh detected Mimikatz accessing lsass.exe and classified it as MITRE T1003.001 — LSASS Memory*

---

### 2. Shuffle — Full workflow execution
![Shuffle Workflow](screenshots/Shuffle-Workflow.png)
*Self-hosted Shuffle workflow: Webhook → hash extraction → VirusTotal → TheHive, all steps succeeding*

---

### 3. VirusTotal enrichment result
![VirusTotal Result](screenshots/Virustotal-Fetching-Results.png)
*VirusTotal returned 61/67 engine detections for the extracted hash*

---

### 4. TheHive — Auto-created investigation case
![TheHive Case](screenshots/04-thehive-case.png)
*TheHive alert created automatically, including Wazuh rule data and VirusTotal detection ratio*

---

### 5. Architecture diagram
![Architecture](screenshots/05-architecture.png)
*Full pipeline architecture across local PC and two Azure VMs*

---

## Skills Demonstrated

### Security Operations
- SIEM deployment, rule tuning, and alert triage
- Windows event log and Sysmon telemetry analysis
- MITRE ATT&CK technique mapping (T1003 — Credential Dumping)
- IOC enrichment and threat intelligence workflows
- Incident case management

### Technical & Engineering
- Linux server administration (Ubuntu 24.04)
- Docker and Docker Compose — multi-service self-hosted deployments
- REST API integration and authentication (VirusTotal, TheHive)
- SOAR workflow design, including regex-based data transformation
- Cloud infrastructure provisioning on Microsoft Azure (VMs, NSGs, VNets)
- Production-style incident debugging: disk space exhaustion, service crash diagnosis via `journalctl`, network connectivity troubleshooting, API permission errors
- SSH remote administration and Azure Cloud Shell

### Tools & Technologies
`Wazuh` `Shuffle SOAR` `TheHive` `VirusTotal API` `Sysmon` `Docker` `Microsoft Azure` `VirtualBox` `Ubuntu` `Windows 10` `REST APIs` `JSON` `XML` `Linux CLI` `Regex`

---

## How to Replicate

**Prerequisites**
- Microsoft Azure account (free trial credit)
- PC with 8 GB+ RAM (for the Windows VirtualBox client)
- VirusTotal account + free API key

**Estimated time:** 5–8 hours (including troubleshooting)
**Estimated cost:** $0 (within Azure free trial credit)

**High-level setup steps:**
1. Provision two Azure VMs (Ubuntu 24.04) — one for Wazuh, one for Shuffle + TheHive
2. Install Wazuh via the official installer script
3. Deploy Shuffle and TheHive as Docker containers on the second VM
4. Create a Windows 10 VM in VirtualBox, install Sysmon and the Wazuh Agent
5. Configure `ossec.conf` to forward alerts to the Shuffle webhook
6. Build the Shuffle workflow: Webhook → regex hash extraction → VirusTotal → TheHive
7. Validate with a Mimikatz credential-dumping test

---

## License

MIT License — feel free to use this as a reference for your own SOC homelab.

---

<div align="center">
<br>
<i>Built as part of a cybersecurity and networking portfolio.</i><br>
<i>Open to SOC Analyst, Security Analyst, and Network Security roles.</i>
</div>
