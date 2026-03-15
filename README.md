# 🛡️ Mini Home SOC Lab

> A fully functional Security Operations Center (SOC) lab built on a Windows PC using Hyper-V — covering endpoint telemetry, log forwarding, SIEM ingestion, and threat detection.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Lab Environment](#lab-environment)
- [Part 1 — Hyper-V Setup & VM Provisioning](#part-1--hyper-v-setup--vm-provisioning)
- [Part 2 — OPNsense Firewall](#part-2--opnsense-firewall)
- [Part 3 — Splunk Enterprise on Ubuntu](#part-3--splunk-enterprise-on-ubuntu)
- [Part 4 — Windows Server Endpoint Telemetry](#part-4--windows-server-endpoint-telemetry)
- [Part 5 — Splunk Universal Forwarder (Windows)](#part-5--splunk-universal-forwarder-windows)
- [Part 6 — Splunk Universal Forwarder (Ubuntu)](#part-6--splunk-universal-forwarder-ubuntu)
- [Part 7 — Splunk Add-Ons & CIM Normalization](#part-7--splunk-add-ons--cim-normalization)
- [Part 8 — Detection Use Cases & SPL Queries](#part-8--detection-use-cases--spl-queries)
- [Part 9 — Attack Simulation & Testing](#part-9--attack-simulation--testing)
- [Part 10 — Data Models & Coverage](#part-10--data-models--coverage)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

---

## Overview

This project documents how I built a home SOC lab from scratch using only a Windows PC and Hyper-V. The goal is to practice real-world threat detection skills across Windows endpoint telemetry, Linux log forwarding, and network-layer visibility.

**Core detection use cases covered:**
- Failed logon / brute force (Event ID 4625)
- Privilege escalation (Event ID 4672)
- Process creation with command-line visibility (Event ID 4688)
- PowerShell script block execution (Event ID 4104)
- Sysmon process creation and network connections (Event IDs 1, 3)
- Account creation and group changes (Event IDs 4720, 4740)
- SSH brute force on Linux (`auth.log`)
- C2 / beaconing / data exfiltration *(via OPNsense network telemetry — in progress)*

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Windows Host (Hyper-V)                  │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │  OPNsense VM │    │  Windows     │    │  Ubuntu VM   │  │
│  │  (Firewall)  │    │  Server VM   │    │  (Splunk     │  │
│  │              │    │              │    │   Enterprise)│  │
│  │  WAN ──► LAN │    │  Sysmon +    │    │              │  │
│  │  (vSwitch)   │    │  UForwarder  │───►│  Port 9997   │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│                            │                    ▲           │
│                            └────────────────────┘           │
│                         Log Pipeline (TCP 9997)              │
└─────────────────────────────────────────────────────────────┘
```

**Data Flow:**
`Windows Server` → `Splunk Universal Forwarder` → `Splunk Enterprise (Ubuntu)` → `Dashboards & Alerts`

---

## Lab Environment

| Component | Details |
|-----------|---------|
| **Host OS** | Windows 10/11 |
| **Hypervisor** | Hyper-V (built-in) |
| **Firewall VM** | OPNsense (FreeBSD-based, amd64 dvd image) |
| **SIEM VM** | Ubuntu LTS + Splunk Enterprise |
| **Endpoint VM** | Windows Server 2019/2022 |
| **Log Agent** | Splunk Universal Forwarder |
| **Endpoint Telemetry** | Sysmon (SwiftOnSecurity config) |
| **SIEM** | Splunk Enterprise (free license — up to 500MB/day) |

---

## Part 1 — Hyper-V Setup & VM Provisioning

### Enable Hyper-V

Open PowerShell as Administrator:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

Reboot when prompted.

### Virtual Switch Configuration

Open **Hyper-V Manager → Virtual Switch Manager** and create the following switches:

| Switch Name | Type | Purpose |
|-------------|------|---------|
| `External-WAN` | External | OPNsense WAN (connects to physical NIC) |
| `Internal-LAN` | Internal | Lab internal network (OPNsense LAN, Windows Server, Ubuntu) |

> ⚠️ **Important:** Manage vEthernet adapters through Hyper-V Manager's Virtual Switch Manager only — not directly via Windows Network Connections. Modifying them there can break Hyper-V networking.

### VM Specifications

| VM | Generation | RAM | vCPU | Disk | Network Adapters |
|----|-----------|-----|------|------|-----------------|
| OPNsense | Gen 1 | 2 GB | 2 | 16 GB | 2 (WAN + LAN) |
| Ubuntu / Splunk | Gen 2 | 4 GB | 2 | 60 GB | 1 (Internal-LAN) |
| Windows Server | Gen 2 | 4 GB | 2 | 60 GB | 1 (Internal-LAN) |

> **OPNsense must be Generation 1** — FreeBSD does not support Hyper-V Gen 2 Secure Boot by default.

---

## Part 2 — OPNsense Firewall

OPNsense is a standalone FreeBSD-based OS. It runs as its own VM — do **not** try to install it inside another OS.

### Download

1. Go to [https://opnsense.org/download/](https://opnsense.org/download/)
2. Select: **Architecture: amd64** | **Image type: dvd** | **Mirror: closest to you**
3. Download the `.iso.bz2` file

> ⚠️ **Windows' built-in extractor does NOT support `.bz2` files.** Use [7-Zip](https://www.7-zip.org/) to extract the `.iso`.

```
Right-click the .bz2 file → 7-Zip → Extract Here
```

### Create OPNsense VM in Hyper-V

1. New VM → **Generation 1**
2. Assign **2 GB RAM** (minimum)
3. Add **two network adapters**:
   - Adapter 1 → `External-WAN`
   - Adapter 2 → `Internal-LAN`
4. Mount the extracted `.iso` to the DVD drive
5. Boot and follow the installer

### OPNsense Interface Assignment

During first boot, assign interfaces when prompted:

| Role | Hyper-V vSwitch | OPNsense Interface |
|------|-----------------|--------------------|
| WAN | External-WAN | `hn0` (or `vtnet0`) |
| LAN | Internal-LAN | `hn1` (or `vtnet1`) |

After installation, access the web GUI at:
```
http://192.168.1.1   (default LAN IP)
```
Default credentials: `root` / `opnsense`

### Key OPNsense Configurations

- **LAN DHCP**: Enabled — assigns IPs to Windows Server and Ubuntu VMs
- **Firewall Rules**: Allow LAN → Any for lab traffic
- **Logging**: Enable firewall log exports for future Splunk ingestion (network telemetry)
- **Syslog**: Configure under `System → Logging → Remote` to forward to Splunk (UDP 514)

---

## Part 3 — Splunk Enterprise on Ubuntu

### Prepare Ubuntu VM

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget curl -y
```

### Download & Install Splunk Enterprise

1. Get the download link from [splunk.com](https://www.splunk.com/en_us/download/splunk-enterprise.html) (free account required)

```bash
# Download (replace with actual wget link from Splunk)
wget -O splunk.tgz '<your-splunk-wget-link>'

# Verify file
ls -lh

# Extract
sudo tar -xvf splunk.tgz -C /opt/
```

### Create Splunk User & Set Permissions

```bash
sudo useradd -m -d /home/splunk -s /bin/bash splunk
sudo passwd splunk
sudo chown -R splunk:splunk /opt/splunk
```

### Start Splunk

```bash
su - splunk
/opt/splunk/bin/splunk start --accept-license
# Set admin username and password when prompted
```

### Access Splunk Web

```
http://<ubuntu-vm-ip>:8000
```

### Enable Splunk to Receive Forwarded Logs (Port 9997)

In Splunk Web: **Settings → Forwarding and Receiving → Configure Receiving → Add New → Port 9997**

Or via CLI:
```bash
/opt/splunk/bin/splunk enable listen 9997 -auth admin:<password>
```

### Create Indexes

In Splunk Web: **Settings → Indexes → New Index**

| Index Name | Purpose |
|------------|---------|
| `windows` | Windows Security, PowerShell, System logs |
| `sysmon` | Sysmon endpoint telemetry |
| `main` | Linux logs (auth, syslog, kern) |

### Splunk Management Ports (if Enterprise and Forwarder on same host)

| Service | Port |
|---------|------|
| Splunk Enterprise Web | 8000 |
| Splunk Enterprise Management | 8089 |
| Splunk Forwarder Management | 8090 |
| Log Receiving | 9997 |

---

## Part 4 — Windows Server Endpoint Telemetry

### 4a. Install Sysmon

1. Download from: [https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
2. Extract to `C:\Tools\Sysmon`
3. Open **Command Prompt as Administrator**

```cmd
cd C:\Tools\Sysmon
sysmon64.exe -accepteula -i
```

### 4b. Apply SwiftOnSecurity Configuration

Download the config from [https://github.com/SwiftOnSecurity/sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config):

```cmd
# Copy sysmonconfig-export.xml to C:\Tools\Sysmon, then:
sysmon64.exe -c sysmonconfig-export.xml

# Verify config loaded
sysmon64.exe -c
```

> Sysmon logs live at: `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`

**Key Sysmon Event IDs:**

| Event ID | Description |
|----------|-------------|
| 1 | Process Create |
| 3 | Network Connection |
| 7 | Image Loaded |
| 11 | File Created |
| 12/13 | Registry Events |
| 22 | DNS Query |

### 4c. Enable Windows Security Auditing

Open `gpedit.msc` (Win + R):

Navigate to: `Computer Configuration → Windows Settings → Security Settings → Advanced Audit Policy Configuration`

Enable the following:

**Account Logon:**
- ✅ Audit Credential Validation (Success, Failure)

**Logon/Logoff:**
- ✅ Audit Logon (Success, Failure)
- ✅ Audit Logoff (Success)
- ✅ Audit Special Logon (Success)
- ✅ Audit Other Logon/Logoff Events (Success, Failure)

**Account Management:**
- ✅ Audit User Account Management (Success, Failure)
- ✅ Audit Security Group Management (Success, Failure)

**Policy Change:**
- ✅ Audit Authentication Policy Change (Success, Failure)

**Privilege Use:**
- ✅ Audit Sensitive Privilege Use (Success, Failure)

**Detailed Tracking:**
- ✅ Audit Process Creation (Success)

Apply changes:
```powershell
gpupdate /force

# Verify audit policy
auditpol /get /category:*
```

### 4d. Enable Command-Line Logging (Event ID 4688)

In `gpedit.msc`:

```
Computer Configuration → Administrative Templates → System → Audit Process Creation
→ Enable: "Include command line in process creation events"
```

This adds full command-line arguments to **Event ID 4688** in the Security log.

### 4e. Enable PowerShell Logging

In `gpedit.msc`:

```
Computer Configuration → Administrative Templates → Windows Components → Windows PowerShell
```

Enable all three:
- ✅ Turn on Module Logging
- ✅ Turn on Script Block Logging
- ✅ Turn on PowerShell Transcription

**Key PowerShell Event IDs:**

| Event ID | Description | Log |
|----------|-------------|-----|
| 4103 | Module Logging | PowerShell/Operational |
| 4104 | Script Block Logging | PowerShell/Operational |
| 4105/4106 | Script Block Start/Stop | PowerShell/Operational |

> ⚠️ **Caution:** Script block logging captures all PowerShell content in plaintext. Consider **ProtectedEventLogging** if sensitive scripts are run in the environment.

---

## Part 5 — Splunk Universal Forwarder (Windows)

### Install

Download the Windows Universal Forwarder `.msi` from [splunk.com](https://www.splunk.com/en_us/download/universal-forwarder.html) and install it.

During installation, point it at your Splunk Enterprise server IP on port `9997`.

### Configure inputs.conf

Location: `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`

```ini
# Windows Security Log (logons, account management, privilege use, process creation)
[WinEventLog://Security]
disabled = 0
renderXml = false
index = windows
sourcetype = WinEventLog

# PowerShell Operational Log (Script Block Logging - Event ID 4104)
[WinEventLog://Microsoft-Windows-PowerShell/Operational]
disabled = 0
renderXml = true
index = windows
sourcetype = WinEventLog

# Sysmon Operational Log
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
renderXml = false
index = sysmon
sourcetype = WinEventLog
```

> **Note on renderXml:** `false` = plain text (easier to start with). `true` = XML format (better for CIM field extraction with modern add-ons). If you switch to `true`, allow/deny filters require `$XmlRegex` syntax.

### Configure outputs.conf

Location: `C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf`

```ini
[tcpout]
defaultGroup = default-autolb-group

[tcpout:default-autolb-group]
server = <splunk-ubuntu-vm-ip>:9997

[tcpout-server://<splunk-ubuntu-vm-ip>:9997]
```

### Restart the Forwarder

```cmd
# From elevated command prompt
"C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe" restart
```

### Verify Forwarding

```powershell
# Check that port 9997 is reachable
Test-NetConnection -ComputerName <splunk-ip> -Port 9997
```

---

## Part 6 — Splunk Universal Forwarder (Ubuntu)

### Check Architecture

```bash
uname -m
# x86_64 = 64-bit AMD/Intel
# aarch64 or arm64 = ARM
```

### Install

```bash
# Download (replace with actual wget link from Splunk)
wget -O splunkforwarder.deb '<your-forwarder-wget-link>'

# Install
sudo dpkg -i splunkforwarder.deb

# Set ownership
sudo chown -R splunk:splunk /opt/splunkforwarder

# Start and accept license
/opt/splunkforwarder/bin/splunk start --accept-license
# Create admin credentials when prompted
```

### Add Forward Server

```bash
/opt/splunkforwarder/bin/splunk add forward-server <splunk-enterprise-ip>:9997
```

### Configure outputs.conf

```bash
sudo nano /opt/splunkforwarder/etc/system/local/outputs.conf
```

```ini
[tcpout:default-autolb-group]
server = <splunk-enterprise-ip>:9997
```

### Configure inputs.conf

```bash
sudo nano /opt/splunkforwarder/etc/system/local/inputs.conf
```

```ini
[monitor:///var/log/auth.log]
disabled = false
index = main
sourcetype = linux_secure
host = <ubuntu-hostname>

[monitor:///var/log/syslog]
disabled = false
index = main
sourcetype = linux_syslog
host = <ubuntu-hostname>

[monitor:///var/log/kern.log]
disabled = false
index = main
sourcetype = linux_kernel
host = <ubuntu-hostname>
```

### Verify IP (for outputs.conf)

```bash
ip a
```

---

## Part 7 — Splunk Add-Ons & CIM Normalization

Install the following add-ons on the **Splunk Enterprise VM** via Splunk Web (`Apps → Find More Apps`):

| Add-On | Purpose |
|--------|---------|
| **Splunk Add-On for Microsoft Windows** | Maps Windows event logs to CIM fields |
| **Splunk Add-On for Sysmon** | CIM-compatible field extraction for Sysmon events |
| **Splunk Common Information Model (CIM)** | Backbone data models for portable detections |
| **Splunk Security Essentials (SSE)** | Explore security use cases and guided detections |

> CIM is what allows you to write detection queries that work regardless of the raw log format — it normalizes fields like `src_ip`, `user`, `process` across all sources.

### Validate Ingestion (Splunk Web → Search)

```spl
# Sysmon data
index=sysmon sourcetype=WinEventLog

# Windows Security Events (failed logons)
index=windows EventCode=4625

# PowerShell Script Block Logging
index=windows EventCode=4104

# Linux auth logs
index=main sourcetype=linux_secure
```

---

## Part 8 — Detection Use Cases & SPL Queries

### Failed Logon / Brute Force

```spl
index=windows EventCode=4625
| stats count by Account_Name, src_ip, host
| where count > 5
| sort -count
```

### Successful Logon After Multiple Failures

```spl
index=windows (EventCode=4625 OR EventCode=4624)
| stats count(eval(EventCode=4625)) as failures,
         count(eval(EventCode=4624)) as successes by Account_Name
| where failures > 5 AND successes > 0
```

### Account Lockout

```spl
index=windows EventCode=4740
| table _time, Account_Name, src_ip, host
```

### Privilege Escalation (Special Privileges Assigned)

```spl
index=windows EventCode=4672
| stats count by Account_Name, Privileges, host
| where NOT Account_Name="SYSTEM"
```

### Process Creation with Command Line (Event ID 4688)

```spl
index=windows EventCode=4688
| table _time, New_Process_Name, Process_Command_Line, Creator_Process_Name, Account_Name
| search Process_Command_Line="*powershell*" OR Process_Command_Line="*cmd*" OR Process_Command_Line="*wscript*"
```

### Sysmon — Process Creation

```spl
index=sysmon EventCode=1
| table _time, Image, CommandLine, ParentImage, User, host
```

### Sysmon — Suspicious Network Connections

```spl
index=sysmon EventCode=3
| table _time, Image, DestinationIp, DestinationPort, User, host
| where NOT DestinationIp="127.0.0.1"
```

### PowerShell Script Block Logging

```spl
index=windows EventCode=4104
| table _time, ScriptBlockText, Path, host
| search ScriptBlockText="*Invoke-*" OR ScriptBlockText="*EncodedCommand*" OR ScriptBlockText="*DownloadString*"
```

### Account Creation

```spl
index=windows EventCode=4720
| table _time, Sam_Account_Name, Subject_Account_Name, host
```

### Service Installation (Persistence)

```spl
index=windows EventCode=7045
| table _time, Service_Name, Service_File_Name, Service_Account, host
```

### Linux SSH Brute Force

```spl
index=main sourcetype=linux_secure "Failed password"
| rex field=_raw "Failed password for (?<user>\S+) from (?<src_ip>\S+)"
| stats count by user, src_ip
| where count > 10
| sort -count
```

---

## Part 9 — Attack Simulation & Testing

### ⚠️ Important: Always snapshot your VMs before running any simulation. Revert when done.

### Option A: Atomic Red Team (Recommended for Repeatability)

Atomic Red Team provides ATT&CK-aligned tests you can run with PowerShell.

```powershell
# Install the module (on Windows Server VM)
Install-Module -Name invoke-atomicredteam -Scope CurrentUser
Import-Module invoke-atomicredteam

# Run a specific test (example: T1059.001 - PowerShell)
Invoke-AtomicTest T1059.001 -GetPrereqs
Invoke-AtomicTest T1059.001
```

### Option B: Benign Log Generators (Simple & Safe)

**Simulate Failed Logons (Event ID 4625):**
From the Windows Server console, attempt to log in with a wrong password 5-10 times using an invalid account. No tools needed — this creates realistic 4625 events.

**Simulate Privilege Escalation (Event ID 4672):**
```powershell
# Add a standard user to local admins (lab only — revert after!)
net localgroup Administrators testuser /add
```
Watch for Event IDs 4728 (group add) and 4672 (special privilege logon).

**Simulate Suspicious PowerShell (Event ID 4104):**
```powershell
# Encoded command - generates detectable patterns SOCs alert on
powershell -EncodedCommand "V2hvYW1p"
# (Base64 of "Whoami" - harmless)
```

**Simulate Brute Force on Linux:**
```bash
# From another VM, attempt SSH logins with wrong password
ssh wronguser@<ubuntu-ip>  # repeat 10+ times
```

### Validate End-to-End Pipeline

The smoke test for lab readiness:
1. Generate a known event (e.g., failed logon)
2. Confirm it appears in Splunk (`index=windows EventCode=4625`)
3. Verify the alert/dashboard reflects it

If step 2 fails, troubleshoot in this order:
1. Is the `inputs.conf` stanza present and correct?
2. Does the UF service account have read access to the Security log?
3. Is port 9997 open between the Windows Server and Ubuntu VMs?
4. Is the audit policy actually generating the events? (`wevtutil qe Security /c:5 /rd:true /f:text`)

---

## Part 10 — Data Models & Coverage

### Priority Data Models (CIM)

| Model | Priority | Detects |
|-------|----------|---------|
| **Authentication** | 🔴 High | 4624/4625, Brute Force, Password Spraying, Lateral Movement |
| **Endpoint** | 🔴 High | Process Creation (4688), Command Line, Parent/Child Analysis, Malware |
| **Network Traffic** | 🟡 Medium | C2 Detection, Data Exfiltration, Beaconing (requires OPNsense) |
| **Change** | 🟡 Medium | Account Creation (4720), Privilege Changes, Group Membership |

### Detection Coverage Score

| Area | Coverage | Status |
|------|----------|--------|
| Windows Authentication | Excellent | ✅ |
| Windows Endpoint Telemetry (Sysmon) | Excellent | ✅ |
| PowerShell Monitoring | Good | ✅ |
| Linux Authentication | Good | ✅ |
| Linux System Logs | Good | ✅ |
| Network Telemetry (OPNsense) | In Progress | 🔄 |
| Windows Defender Logs | Not Yet Added | ⬜ |
| RDP Monitoring (4624 LogonType=10) | Not Yet Added | ⬜ |

### Recommended Summary Retention

| Duration | Recommendation |
|----------|---------------|
| 7 days | Good for a lab |
| 14 days | Better |
| 30 days | Best (if disk space allows) |

---

## Troubleshooting

### OPNsense ISO Won't Extract
- Windows' built-in extractor doesn't support `.bz2` — use **7-Zip**
- If the archive appears corrupted, try a different mirror on the OPNsense download page

### Splunk Not Receiving Events
Run this diagnostic sequence in order:
1. Confirm `inputs.conf` stanza exists and correct event log channel name is used
2. Confirm UF service account has permissions to read the Security log
3. Test port 9997 connectivity: `Test-NetConnection -ComputerName <splunk-ip> -Port 9997`
4. Confirm audit policy is generating events locally: `wevtutil qe Security /c:5 /rd:true /f:text`
5. Check UF logs: `C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log`

### Missing Event ID 4625 / 4771 in Splunk
- Verify `Audit Logon` is set to **Success, Failure** in Advanced Audit Policy
- Run `gpupdate /force` after any policy change
- Check `auditpol /get /category:*` to confirm policy is applied

### Hyper-V vEthernet Adapter Issues
- Manage vEthernet adapters **only** through Hyper-V Manager → Virtual Switch Manager
- Do not delete or modify them through Windows Network Connections — this can break VM networking

### OPNsense WAN/LAN Interface Assignment
- OPNsense assigns interface names like `hn0`, `hn1` for Hyper-V virtual NICs
- Verify which adapter is which using the MAC address visible in Hyper-V Manager

---

## Repository Structure

```
mini-soc-lab/
├── README.md                          ← You are here
├── configs/
│   ├── splunk-forwarder/
│   │   ├── inputs.conf                ← Windows UF inputs (Security, Sysmon, PowerShell)
│   │   ├── outputs.conf               ← Windows UF forwarding config
│   │   ├── inputs_ubuntu.conf         ← Ubuntu UF inputs (auth, syslog, kern)
│   │   └── outputs_ubuntu.conf        ← Ubuntu UF forwarding config
│   ├── splunk-enterprise/
│   │   └── indexes.conf               ← Index definitions (windows, sysmon, main)
│   ├── sysmon/
│   │   └── sysmonconfig-export.xml    ← SwiftOnSecurity config (download from GitHub)
│   └── opnsense/
│       └── firewall-rules.md          ← OPNsense LAN rule notes
├── detection-queries/
│   ├── authentication.spl             ← Brute force, lockout, logon queries
│   ├── endpoint.spl                   ← Process creation, Sysmon queries
│   ├── powershell.spl                 ← Script block and encoded command queries
│   ├── privilege-escalation.spl       ← 4672, group change queries
│   └── linux.spl                      ← SSH brute force, auth queries
└── docs/
    ├── 01-hyperv-setup.md
    ├── 02-opnsense-install.md
    ├── 03-splunk-enterprise-install.md
    ├── 04-windows-telemetry.md
    ├── 05-splunk-forwarder-windows.md
    ├── 06-splunk-forwarder-ubuntu.md
    ├── 07-add-ons-and-cim.md
    ├── 08-detection-testing.md
    └── 09-data-models.md
```

---

## Resources

| Resource | Link |
|----------|------|
| Sysmon Download | https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon |
| SwiftOnSecurity Sysmon Config | https://github.com/SwiftOnSecurity/sysmon-config |
| Olaf Hartong sysmon-modular | https://github.com/olafhartong/sysmon-modular |
| Atomic Red Team | https://github.com/redcanaryco/atomic-red-team |
| Invoke-AtomicRedTeam | https://github.com/redcanaryco/invoke-atomicredteam |
| OPNsense Download | https://opnsense.org/download/ |
| Splunk Enterprise | https://www.splunk.com/en_us/download/splunk-enterprise.html |
| Splunk Universal Forwarder | https://www.splunk.com/en_us/download/universal-forwarder.html |
| Splunk Add-On for Microsoft Windows | https://splunkbase.splunk.com/app/742 |
| Splunk Add-On for Sysmon | https://splunkbase.splunk.com/app/5709 |
| Splunk CIM | https://splunkbase.splunk.com/app/1621 |
| Splunk Security Essentials | https://splunkbase.splunk.com/app/3435 |
| MITRE ATT&CK | https://attack.mitre.org |
| 7-Zip | https://www.7-zip.org |

---

## License

This project is for educational purposes. All configurations are intended for use in isolated lab environments only.

---

*Built with Hyper-V on Windows | Splunk Enterprise + Universal Forwarder | OPNsense | Sysmon*
