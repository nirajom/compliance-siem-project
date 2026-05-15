#  SIEM-Based Compliance Monitoring System

A complete Security Information and Event Management (SIEM) system built using **Wazuh 4.7.5** on a local KVM/QEMU virtualization lab. This project collects real-time security logs from endpoints, detects threats, maps alerts to compliance frameworks (**PCI-DSS**, **ISO 27001**, **CIS Benchmarks**, **MITRE ATT&CK**), and visualizes everything through a centralized dashboard.

---

##  Table of Contents

- [Project Overview](#project-overview)
- [Architecture](#architecture)
- [Lab Environment](#lab-environment)
- [Step 1 — Host System Setup (Parrot OS)](#step-1--host-system-setup-parrot-os)
- [Step 2 — Ubuntu Server VM Creation](#step-2--ubuntu-server-vm-creation)
- [Step 3 — Wazuh Server Installation](#step-3--wazuh-server-installation)
- [Step 4 — Wazuh Agent Installation (Parrot OS)](#step-4--wazuh-agent-installation-parrot-os)
- [Step 5 — Custom Compliance Rules](#step-5--custom-compliance-rules)
- [Step 6 — Attack Simulations](#step-6--attack-simulations)
- [Step 7 — Dashboard & Compliance Monitoring](#step-7--dashboard--compliance-monitoring)
- [Compliance Mapping](#compliance-mapping)
- [Screenshots](#screenshots)
- [Viva / Interview Q&A](#viva--interview-qa)
- [Future Scope](#future-scope)
- [Author](#author)

---

## Project Overview

### Problem Statement

> The organization requires compliance readiness. You must map logs and alerts to compliance frameworks such as PCI-DSS or ISO standards and produce a dashboard.

### What This Project Does

1. **Collects logs** from endpoints (authentication, system, file changes, network events)
2. **Detects threats** in real-time (brute force, port scans, malware, privilege escalation)
3. **Maps every alert** to specific compliance controls (PCI-DSS, ISO 27001, NIST, CIS)
4. **Visualizes** everything on a centralized Wazuh dashboard
5. **Generates compliance reports** showing which standards are being met or violated

### Workflow

```
Parrot OS (Endpoint with Wazuh Agent)
        │
        │ Logs forwarded via Port 1514 (Encrypted)
        ▼
Ubuntu Server VM (Wazuh Manager)
        │
        ├── Rule Engine (analyzes logs, applies compliance rules)
        ├── Wazuh Indexer (stores and indexes all events)
        └── Wazuh Dashboard (visualization + compliance modules)
                │
                ▼
        PCI-DSS / ISO 27001 / MITRE ATT&CK / CIS Reports
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    PARROT OS HOST                        │
│                  (KVM/QEMU Hypervisor)                   │
│                                                          │
│   ┌──────────────┐         ┌──────────────────────────┐ │
│   │  Wazuh Agent │         │   Ubuntu Server 26.04    │ │
│   │  (Parrot OS) │────────►│                          │ │
│   │              │ Port    │  ┌──────────────────┐    │ │
│   │ Collects:    │ 1514    │  │  Wazuh Manager   │    │ │
│   │ - auth.log   │         │  │  (Rule Engine)   │    │ │
│   │ - syslog     │         │  └────────┬─────────┘    │ │
│   │ - sudo logs  │         │           │              │ │
│   │ - FIM events │         │  ┌────────▼─────────┐    │ │
│   └──────────────┘         │  │  Wazuh Indexer   │    │ │
│                            │  │  (OpenSearch)    │    │ │
│   ┌──────────────┐         │  └────────┬─────────┘    │ │
│   │   Kali VM    │         │           │              │ │
│   │  (Attacker)  │────────►│  ┌────────▼─────────┐    │ │
│   │              │ Attacks │  │ Wazuh Dashboard  │    │ │
│   │ Tools:       │         │  │ (Port 443/HTTPS) │    │ │
│   │ - Hydra      │         │  └──────────────────┘    │ │
│   │ - Nmap       │         │                          │ │
│   └──────────────┘         │  IP: 192.168.122.107     │ │
│                            └──────────────────────────┘ │
│                                                          │
│   Network: virbr0 (192.168.122.0/24) — NAT mode        │
└─────────────────────────────────────────────────────────┘
```

---

## Lab Environment

| Component | Details |
|-----------|---------|
| **Host OS** | Parrot Security 7.2 |
| **Hypervisor** | KVM/QEMU + libvirt (virt-manager) |
| **Wazuh Server VM** | Ubuntu Server 26.04 LTS, 4GB RAM, 2 vCPU, 40GB disk |
| **Wazuh Version** | 4.7.5 (Manager + Indexer + Dashboard) |
| **Agent** | Parrot OS host (self-monitoring) |
| **Attacker VM** | Kali Linux (for attack simulation) |
| **Network** | virbr0 NAT — 192.168.122.0/24 |
| **Wazuh Server IP** | 192.168.122.107 |
| **Dashboard URL** | https://192.168.122.107 |

---

## Step 1 — Host System Setup (Parrot OS)

Parrot OS is our base operating system running on physical hardware. It serves two roles:
- **KVM hypervisor** — runs all VMs
- **Wazuh agent endpoint** — gets monitored by the SIEM

### Prerequisites Verified

```bash
# Check available RAM (need 10GB+ total for all VMs)
free -h

# Check disk space (need 50GB+ free)
df -h ~

# Verify KVM is working
virsh list --all
```

### KVM/QEMU Already Installed

```bash
sudo apt install qemu-kvm libvirt-daemon-system virt-manager -y
sudo systemctl enable --now libvirtd
```

---

## Step 2 — Ubuntu Server VM Creation

We need a dedicated VM to run the Wazuh server. Ubuntu Server is lightweight and perfect for this.

### 2.1 Download Ubuntu Server ISO

```bash
cd ~/Downloads
wget https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso
```

### 2.2 Create VM using virt-manager

1. Open **virt-manager** (Virtual Machine Manager)
2. Click **Create a new virtual machine**
3. Select **Local install media (ISO image)**
4. Browse and select the Ubuntu Server ISO
5. Configure resources:

| Setting | Value |
|---------|-------|
| RAM | 4096 MB (4GB) — **minimum required for Wazuh** |
| CPUs | 2 cores |
| Disk | 40 GB (qcow2 format) |
| Network | NAT (default virbr0) |
| Name | `wazuh-server` |

6. Start installation

### 2.3 Ubuntu Server Installation Options

During installation, we selected:

| Screen | Selection |
|--------|-----------|
| Language | English |
| Keyboard | Default |
| Network | DHCP (auto-assigned 192.168.122.107) |
| Storage | Use entire disk |
| Profile | Username: `om`, Hostname: `wazuh-server` |
| SSH |  Install OpenSSH server |
| Ubuntu Pro |  Skip for now (not needed for lab) |
| Featured Snaps |  None selected (saves RAM and disk) |

### 2.4 Post-Installation Verification

```bash
# Login to the VM
ssh om@192.168.122.107

# Verify internet connectivity
ping google.com

# Check system resources
free -h
df -h /
```

**Screenshot:** [Ubuntu Server VM running and accessible via SSH](screenshots/01-ubuntu-server-running.png)

---

## Step 3 — Wazuh Server Installation

Wazuh all-in-one installer sets up three components on the same machine:
- **Wazuh Manager** — receives and analyzes logs
- **Wazuh Indexer** — stores events (OpenSearch-based)
- **Wazuh Dashboard** — web UI for visualization

### 3.1 Run the Installer

```bash
# SSH into Ubuntu Server VM
ssh om@192.168.122.107

# Update system
sudo apt update && sudo apt upgrade -y

# Download and run Wazuh installer
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

This takes 10-15 minutes. At the end, it prints admin credentials:

```
INFO: --- Summary ---
INFO: You can access the web interface https://192.168.122.107
    User: admin
    Password: <generated-password>
```

>  **Save these credentials immediately** — they are shown only once.

### 3.2 Verify All Services

```bash
sudo systemctl status wazuh-manager    # Should be: active (running)
sudo systemctl status wazuh-indexer    # Should be: active (running)
sudo systemctl status wazuh-dashboard  # Should be: active (running)
```

### 3.3 Access Dashboard

Open browser on Parrot host:

```
https://192.168.122.107
```

Accept the self-signed certificate warning, then login with admin credentials.

**Screenshot:** [Wazuh Dashboard Home Page](screenshots/02-wazuh-dashboard-home.png)

---

## Step 4 — Wazuh Agent Installation (Parrot OS)

The Wazuh agent runs on the endpoint (Parrot OS host) and forwards all security logs to the manager.

### 4.1 Install Agent

```bash
# On Parrot OS host
curl -so wazuh-agent.deb https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.7.0-1_amd64.deb

# Install with manager IP
sudo WAZUH_MANAGER='192.168.122.107' dpkg -i ./wazuh-agent.deb

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
```

### 4.2 Verify Agent Connection

```bash
# Check agent status
sudo systemctl status wazuh-agent

# Should show: active (running) and "Connected to server"
```

### 4.3 Confirm in Dashboard

Go to **Wazuh Dashboard → Agents**

You should see:

| Field | Value |
|-------|-------|
| Agent ID | 001 |
| Status | **Active** (green) |
| IP Address | 192.168.122.1 |
| OS | Parrot Security 7.2 |
| Version | Wazuh v4.7.5 |
| Registration Date | May 15, 2026 |

**Screenshot:** [Agent Active in Dashboard](screenshots/03-agent-active.png)

---

## Step 5 — Custom Compliance Rules

Wazuh has built-in PCI-DSS rules, but we created **custom rules** that map events to both PCI-DSS AND ISO 27001 simultaneously.

### 5.1 Rule File Location

```bash
# SSH into Wazuh server
ssh om@192.168.122.107

# Edit custom rules
sudo nano /var/ossec/etc/rules/local_rules.xml
```

### 5.2 Custom Rules Created

See [`custom-rules/local_rules.xml`](custom-rules/local_rules.xml) for the complete file.

| Rule ID | Trigger | PCI-DSS | ISO 27001 |
|---------|---------|---------|-----------|
| 100001 | Multiple failed SSH logins | 10.2.4 | A.9.4.2 |
| 100002 | Privileged account access | 7.1 | A.9.2.3 |
| 100003 | Critical file modification | 11.5 | A.12.4.1 |
| 100004 | Malware signature detected | 5.1 | A.12.2.1 |
| 100005 | Port scan / reconnaissance | 11.4 | A.13.1.1 |
| 100006 | Unauthorized sudo execution | 8.1.5 | A.9.2.6 |

### 5.3 Apply Rules

```bash
# Restart manager to load new rules
sudo systemctl restart wazuh-manager

# Verify no errors
sudo tail -20 /var/ossec/logs/ossec.log
```

---

## Step 6 — Attack Simulations

We performed multiple attacks from both the Parrot host and Kali VM to generate real security alerts and prove the system detects threats.

### Attack 1: SSH Brute Force

**Purpose:** Test authentication failure detection
**Compliance:** PCI-DSS 10.2.4 / ISO 27001 A.9.4.2

```bash
# From Parrot terminal
ssh fakeuser@localhost
# Enter wrong password 3-4 times
```

**Expected alert:** `sshd: authentication failure` / `Invalid user`

**Screenshot:** [SSH Brute Force Alert in Dashboard](screenshots/04-ssh-brute-force-alert.png)

---

### Attack 2: Port Scanning (Nmap)

**Purpose:** Test network reconnaissance detection
**Compliance:** PCI-DSS 11.4 / ISO 27001 A.13.1.1

```bash
# From Kali VM
sudo nmap -sS -sV 192.168.122.107
```

**Expected alert:** Network scan detected

**Screenshot:** [Port Scan Detection](screenshots/05-port-scan-alert.png)

---

### Attack 3: Privilege Escalation (Sudo Abuse)

**Purpose:** Test unauthorized privilege escalation detection
**Compliance:** PCI-DSS 7.1, 8.1.5 / ISO 27001 A.9.2.3

```bash
# From Parrot terminal (as normal user)
su root              # Wrong password multiple times
sudo cat /etc/shadow # Access sensitive file
```

**Expected alert:** `sudo: authentication failure` / `Privileged access attempt`

**Screenshot:** [Sudo Abuse Alert](screenshots/06-sudo-alert.png)

---

### Attack 4: Malware Detection (EICAR Test)

**Purpose:** Test malware signature detection
**Compliance:** PCI-DSS 5.1 / ISO 27001 A.12.2.1 / NIST SI-3

```bash
# EICAR is an industry-standard safe test file — NOT real malware
echo 'X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*' > /tmp/eicar-test.txt
```

**Expected alert:** Malware/suspicious file detected

**Screenshot:** [EICAR Malware Alert](screenshots/07-malware-alert.png)

---

### Attack 5: File Integrity Tampering

**Purpose:** Test File Integrity Monitoring (FIM)
**Compliance:** PCI-DSS 11.5, 10.5 / ISO 27001 A.12.4.1

```bash
# Modify critical system files
sudo nano /etc/hosts        # Add a comment line
sudo touch /etc/test-tamper  # Create new file in monitored directory
```

**Expected alert:** FIM event showing file hash change

**Screenshot:** [File Integrity Alert](screenshots/08-fim-alert.png)

---

### Attack 6: Rootkit Behavior Simulation

**Purpose:** Test rootcheck / anomaly detection
**Compliance:** ISO 27001 A.12.6.1

```bash
# Create hidden suspicious file
sudo touch /tmp/.hidden-backdoor
sudo chmod +x /tmp/.hidden-backdoor

# Force agent rescan
sudo /var/ossec/bin/agent_control -R 001
```

**Expected alert:** Rootcheck anomaly detection

**Screenshot:** [Rootcheck Alert](screenshots/09-rootcheck-alert.png)

---

## Step 7 — Dashboard & Compliance Monitoring

### 7.1 Wazuh Home Dashboard

The main dashboard shows:
- Total security alerts (176+ detected)
- Active agents
- Alert severity distribution
- Top rule groups (sca, ossec, rootcheck)

**Screenshot:** [Wazuh Home Dashboard](screenshots/10-dashboard-home.png)

### 7.2 Security Events Module

Shows all detected security events with:
- Timestamp
- Alert description
- Rule ID and severity level
- MITRE ATT&CK technique mapping
- PCI-DSS requirement mapping

**Screenshot:** [Security Events Table](screenshots/11-security-events.png)

### 7.3 PCI-DSS Compliance Module

Wazuh automatically maps detected alerts to PCI-DSS requirements:

| PCI-DSS Requirement | Count | Description |
|---------------------|-------|-------------|
| 2.2 | 156 | System configuration standards |
| 2.2.4 | 50 | Security parameter configuration |
| 2.2.3 | 25 | Security features for services |
| 10.6.1 | 20 | Log review automation |
| 2.2.2 | 15 | Necessary services only |

**Screenshot:** [PCI-DSS Compliance Dashboard](screenshots/12-pci-dss.png)

### 7.4 MITRE ATT&CK Module

Maps alerts to MITRE ATT&CK tactics:
- **Defense Evasion** — 1 detection
- More tactics appear as attack simulations trigger alerts

**Screenshot:** [MITRE ATT&CK Mapping](screenshots/13-mitre-attack.png)

### 7.5 SCA (Security Configuration Assessment)

CIS Benchmark scan results for the Parrot agent:

| Benchmark | Passed | Failed | Not Applicable | Score |
|-----------|--------|--------|----------------|-------|
| CIS Debian Linux 7 v1.0.0 | 44 | 101 | 10 | **30%** |

This shows the system needs hardening — which is expected for a lab environment.

**Screenshot:** [CIS Benchmark Results](screenshots/14-sca-benchmark.png)

### 7.6 Service Health Verification

```bash
# Wazuh Manager status
sudo systemctl status wazuh-manager

# Wazuh Agent status  
sudo systemctl status wazuh-agent
```

**Screenshot:** [Service Status Terminal Output](screenshots/15-service-status.png)

---

## Compliance Mapping

Complete mapping of security events to compliance frameworks:

### PCI-DSS Mapping

| PCI-DSS Req | Description | How Wazuh Detects It |
|-------------|-------------|----------------------|
| 2.2 | Configuration standards | CIS Benchmark SCA scan |
| 5.1 | Anti-malware deployment | EICAR / malware rule detection |
| 7.1 | Access control (need-to-know) | Privileged access alerts |
| 8.1.5 | Third-party ID management | Sudo/su abuse detection |
| 10.2.4 | Invalid access attempts logging | SSH failed login alerts |
| 10.5 | Audit trail protection | FIM on log files |
| 10.6.1 | Automated log review | Dashboard + alert generation |
| 11.4 | IDS/IPS for intrusion detection | Port scan / recon alerts |
| 11.5 | File integrity monitoring | FIM module on /etc, /bin, /sbin |

### ISO 27001:2022 Annex A Mapping

| Control | Description | Wazuh Detection |
|---------|-------------|-----------------|
| A.9.2.3 | Privileged access management | Sudo abuse alerts |
| A.9.2.6 | Access rights adjustment | Unauthorized escalation |
| A.9.4.2 | Secure log-on procedures | Failed auth monitoring |
| A.12.2.1 | Malware controls | Malware signature alerts |
| A.12.4.1 | Event logging | Centralized log collection |
| A.12.6.1 | Vulnerability management | Vulnerability assessment |
| A.13.1.1 | Network controls | Port scan detection |

### MITRE ATT&CK Mapping

| Tactic | Technique | Attack Simulated |
|--------|-----------|------------------|
| Credential Access | T1110 — Brute Force | Hydra SSH brute force |
| Privilege Escalation | T1548 — Abuse Elevation | Unauthorized sudo |
| Reconnaissance | T1046 — Network Scanning | Nmap port scan |
| Defense Evasion | T1036 — Masquerading | EICAR test file |
| Impact | T1565 — Data Manipulation | /etc/hosts modification |
| Persistence | T1547 — Autostart Execution | Hidden file in /tmp |

---

## Screenshots

| # | Screenshot | Description |
|---|-----------|-------------|
| 1 | [Ubuntu Server Running](screenshots/01-ubuntu-server-running.png) | VM created and accessible |
| 2 | [Wazuh Dashboard Home](screenshots/02-wazuh-dashboard-home.png) | Main overview with alert counts |
| 3 | [Agent Active](screenshots/03-agent-active.png) | Parrot agent connected and reporting |
| 4 | [SSH Brute Force Alert](screenshots/04-ssh-brute-force-alert.png) | Authentication failure detected |
| 5 | [Port Scan Alert](screenshots/05-port-scan-alert.png) | Nmap scan detected |
| 6 | [Sudo Abuse Alert](screenshots/06-sudo-alert.png) | Privilege escalation attempt |
| 7 | [Malware Alert](screenshots/07-malware-alert.png) | EICAR test file detected |
| 8 | [FIM Alert](screenshots/08-fim-alert.png) | File integrity change detected |
| 9 | [Rootcheck Alert](screenshots/09-rootcheck-alert.png) | Anomaly detection |
| 10 | [Dashboard Home](screenshots/10-dashboard-home.png) | 176+ total alerts visualized |
| 11 | [Security Events](screenshots/11-security-events.png) | Event table with rule details |
| 12 | [PCI-DSS Compliance](screenshots/12-pci-dss.png) | PCI-DSS requirement mapping |
| 13 | [MITRE ATT&CK](screenshots/13-mitre-attack.png) | Tactic correlation |
| 14 | [SCA Benchmark](screenshots/14-sca-benchmark.png) | CIS Debian score: 30% |
| 15 | [Service Status](screenshots/15-service-status.png) | All services active (running) |

---

## Viva / Interview Q&A

**Q: What is SIEM?**
A: Security Information and Event Management — it aggregates logs from multiple sources, correlates events, detects threats, and generates alerts from a centralized platform.

**Q: Why Wazuh?**
A: Wazuh is open-source, supports agent-based log collection, has built-in compliance modules (PCI-DSS, HIPAA, GDPR), includes FIM, rootcheck, vulnerability detection, and integrates with MITRE ATT&CK — all for free.

**Q: What is PCI-DSS?**
A: Payment Card Industry Data Security Standard — a set of 12 requirements that organizations handling credit card data must follow. Requirement 10 (logging) and 11 (monitoring) are directly addressed by this project.

**Q: What is ISO 27001?**
A: International standard for Information Security Management Systems (ISMS). Annex A contains 93 controls covering access control, cryptography, operations security, etc.

**Q: How does the agent communicate with the manager?**
A: The Wazuh agent encrypts logs and forwards them to the manager over port 1514 (TCP/UDP). The manager decodes, analyzes, and applies rules to generate alerts.

**Q: What is File Integrity Monitoring?**
A: FIM tracks changes to critical files (hash, permissions, ownership). If someone modifies /etc/passwd or /etc/hosts, Wazuh detects the change and generates an alert.

**Q: What is the CIS Benchmark?**
A: Center for Internet Security Benchmarks are industry-accepted system hardening guidelines. Our SCA scan scored 30% (44 passed / 101 failed), indicating the lab system needs hardening.

**Q: Difference between IDS and SIEM?**
A: IDS (like Suricata/Snort) only detects network intrusions. SIEM is broader — it collects all types of logs (auth, system, network, file), correlates them, maps to compliance, and provides dashboards.

**Q: What is your contribution in this project?**
A: I designed the lab architecture, set up the Wazuh SIEM on KVM/QEMU virtualization, wrote custom compliance rules mapping events to both PCI-DSS and ISO 27001 simultaneously, performed real attack simulations, and built the compliance monitoring dashboard.

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| **SIEM Platform** | Wazuh 4.7.5 |
| **Search & Storage** | Wazuh Indexer (OpenSearch) |
| **Dashboard** | Wazuh Dashboard (Kibana fork) |
| **Host OS** | Parrot Security 7.2 |
| **Server OS** | Ubuntu Server 26.04 LTS |
| **Hypervisor** | KVM/QEMU + libvirt |
| **Attacker** | Kali Linux |
| **Network** | virbr0 NAT (192.168.122.0/24) |

---

## Future Scope

- **Windows Endpoint Monitoring** — Add Windows VM with Sysmon for advanced process/network logging
- **Suricata IDS Integration** — Network-level intrusion detection feeding into Wazuh
- **GCP Cloud Deployment** — Migrate entire setup to Google Cloud Compute Engine
- **Alerting** — Email and Telegram notifications for critical alerts
- **Threat Intelligence** — VirusTotal API integration for automatic IOC checking
- **YARA Rules** — Custom malware detection signatures
- **EternalBlue Detection** — MS17-010 exploitation detected by SIEM in pivoting scenario

---

## How to Reproduce This Project

```bash
# 1. Create Ubuntu Server VM (4GB RAM, 40GB disk) on KVM
# 2. Install Wazuh all-in-one
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash ./wazuh-install.sh -a

# 3. Install agent on endpoint
sudo WAZUH_MANAGER='<server-ip>' dpkg -i wazuh-agent.deb
sudo systemctl enable --now wazuh-agent

# 4. Copy custom rules
sudo cp custom-rules/local_rules.xml /var/ossec/etc/rules/local_rules.xml
sudo systemctl restart wazuh-manager

# 5. Access dashboard at https://<server-ip>
# 6. Run attack simulations and observe alerts
```


