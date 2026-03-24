# Mini SOC Lab — Home Security Operations Center

A fully functional home SOC lab built on Hyper-V for hands-on detection engineering, threat simulation, and security monitoring. This lab routes all endpoint traffic through a dedicated firewall, collects telemetry from network, endpoint, and authentication layers, and centralizes everything in Splunk Enterprise for analysis and detection.

---

## Architecture

```
                    ┌──────────────────┐
                    │  Internet / ISP  │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  External Switch │
                    │  (WAN - Realtek) │
                    └────────┬─────────┘
                             │ WAN Interface
                    ┌────────▼─────────┐
                    │    OPNsense      │
                    │  Firewall/Router │
                    │  <LAN_GATEWAY>/24 │
                    └────────┬─────────┘
                             │ LAN Interface
                    ┌────────▼─────────┐
                    │  Internal Switch │
                    │    (SOC-LAN)     │
                    └───┬──────────┬───┘
                        │          │
              ┌─────────▼──┐  ┌───▼──────────┐
              │  Windows   │  │   Ubuntu VM   │
              │  Server    │  │   Splunk      │
              │  (DC01)    │  │   Enterprise  │
              │ <WINDOWS_IP> │  │  <SPLUNK_IP>  │
              └────────────┘  └──────────────┘

    Data Flow:
    ─────────
    OPNsense ──── Syslog UDP 5514 ────► Splunk (firewall index)
    Windows  ──── Splunk UF port 9997 ─► Splunk (windows + sysmon indexes)
    Ubuntu   ──── Local log monitoring ─► Splunk (main index)
```

---

## Components

| Component | Role | Platform |
|-----------|------|----------|
| **OPNsense** (Gen 1 VM) | Firewall, router, gateway, network telemetry | FreeBSD on Hyper-V |
| **Windows Server** (DC01) | Domain controller, endpoint telemetry source | Windows Server on Hyper-V |
| **Ubuntu VM** | SIEM — Splunk Enterprise | Ubuntu Desktop LTS on Hyper-V |

---

## Network Design

All lab VMs sit on an isolated internal network (`SOC-LAN`) with OPNsense as the sole gateway. This ensures every packet from any VM traverses the firewall, providing full network visibility.

| Switch | Type | Purpose |
|--------|------|---------|
| `WAN-External` | External (Realtek PCIe NIC) | Connects OPNsense WAN to home router / internet |
| `SOC-LAN` | Internal | Isolated lab LAN — Windows Server and Ubuntu connect here |

| VM | IP | Gateway | DNS |
|----|----|---------|-----|
| OPNsense LAN | `<LAN_GATEWAY>`/24 | — | — |
| OPNsense WAN | DHCP from home router | Home router IP | ISP DNS |
| Windows Server | `<WINDOWS_IP>` (static) | `<LAN_GATEWAY>` | `<LAN_GATEWAY>` |
| Ubuntu (Splunk) | `<SPLUNK_IP>` (static) | `<LAN_GATEWAY>` | `<LAN_GATEWAY>` |

> **Example**: Use any private subnet that doesn't conflict with your home network. A common choice is `10.0.0.0/24` with gateway at `.1`, Windows Server at `.10`, and Splunk at `.20`.

---

## Telemetry Coverage

### Network Layer — OPNsense Firewall Logs
- Syslog forwarding over UDP 5514 to Splunk
- Splunk index: `firewall`
- Sourcetype: `opnsense`
- Visibility: all traffic between VMs and internet, DNS queries, blocked connections

### Endpoint Layer — Sysmon
- Microsoft Sysmon with SwiftOnSecurity configuration
- Forwarded via Splunk Universal Forwarder (port 9997)
- Splunk index: `sysmon`
- Key Event IDs: Process Create (1), Network Connect (3), File Create (11), Registry (12/13/14)

### Authentication & System Layer — Windows Event Logs
- Security, System, Application, PowerShell, Windows Defender, RDP session logs
- Forwarded via Splunk Universal Forwarder (port 9997)
- Splunk index: `windows`
- Key Event IDs: Logon (4624), Failed Logon (4625), Account Lockout (4740), Process Creation (4688), Privilege Escalation (4672), Service Creation (7045), PowerShell Script Block (4104)

### Linux Layer — Ubuntu System Logs
- Local syslog monitoring via Splunk
- Splunk index: `main`
- Sourcetypes: `linux_secure`, `linux_kernel`

---

## Splunk Configuration

### Indexes
| Index | Data Source |
|-------|------------|
| `firewall` | OPNsense syslog |
| `windows` | Windows Security, System, Application, PowerShell, Defender, RDP logs |
| `sysmon` | Sysmon Operational |
| `main` | Ubuntu local logs |

### Inputs

**Splunk Enterprise (Ubuntu) — `/opt/splunk/etc/system/local/inputs.conf`**
```ini
[udp://5514]
sourcetype = opnsense
index = firewall
no_appending_timestamp = true

[monitor:///var/log/syslog]
disabled = false
index = main
sourcetype = linux_syslog
```

**Splunk Universal Forwarder (Windows Server) — `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`**
```ini
[WinEventLog://Security]
disabled = 0
index = windows
renderXml = false
sourcetype = WinEventLog:Security

[WinEventLog://System]
disabled = 0
index = windows
renderXml = false
sourcetype = WinEventLog:System

[WinEventLog://Application]
disabled = 0
index = windows
renderXml = false
sourcetype = WinEventLog:Application

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = sysmon
renderXml = false
sourcetype = WinEventLog:Microsoft-Windows-Sysmon/Operational

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
disabled = 0
index = windows
renderXml = false
sourcetype = WinEventLog:Microsoft-Windows-PowerShell/Operational

[WinEventLog://Microsoft-Windows-Windows Defender/Operational]
disabled = 0
index = windows
renderXml = false
sourcetype = WinEventLog:WindowsDefender

[WinEventLog://Microsoft-Windows-TerminalServices-LocalSessionManager/Operational]
disabled = 0
index = windows
renderXml = false
sourcetype = WinEventLog:LocalSessionManager

[WinEventLog://Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational]
disabled = 0
index = windows
renderXml = false
sourcetype = WinEventLog:RemoteConnectionManager
```

**Splunk Universal Forwarder (Windows Server) — `outputs.conf`**
```ini
[tcpout:default-autolb-group]
server = <SPLUNK_IP>:9997

[tcpout]
sslVersionsForClient = tls1.2
sslVerifyServerCert = false
```

---

## Windows Audit Policies Enabled

Configured via Group Policy (`gpedit.msc`) and Advanced Audit Policy:

- **Account Logon**: Audit Credential Validation (Success, Failure)
- **Logon/Logoff**: Audit Logon (Success, Failure), Audit Logoff (Success), Audit Special Logon (Success), Audit Other Logon/Logoff Events (Success, Failure)
- **Account Management**: Audit User Account Management (Success, Failure), Audit Security Group Management (Success, Failure)
- **Detailed Tracking**: Audit Process Creation (Success) — with command-line logging enabled
- **Policy Change**: Audit Audit Policy Change (Success, Failure), Audit Authentication Policy Change (Success, Failure)
- **Privilege Use**: Audit Sensitive Privilege Use (Success, Failure)
- **PowerShell Logging**: Module Logging, Script Block Logging, Transcription Logging

---

## Detection Coverage

| Area | Coverage | Data Sources |
|------|----------|-------------|
| Windows Authentication | Excellent | Security 4624, 4625, 4740, 4672 |
| Windows Endpoint Telemetry | Excellent | Sysmon Event 1, 3, 11, 12/13/14 |
| Process Execution & Command Line | Excellent | Security 4688 + Sysmon Event 1 |
| PowerShell Monitoring | Good | PowerShell Operational 4104 |
| Network Telemetry | Good | OPNsense firewall logs |
| Linux Authentication | Good | Ubuntu auth.log / syslog |
| RDP Monitoring | Good | LocalSessionManager + RemoteConnectionManager |
| Windows Defender | Good | Defender Operational logs |

---

## Hyper-V Configuration Notes

Key lessons learned during the build:

- **OPNsense must be Gen 1**: FreeBSD is incompatible with Gen 2 Secure Boot; use MBR partition scheme and "Entire Disk" during install
- **Legacy Network Adapters**: OPNsense on Hyper-V Gen 1 only reliably detects Legacy Network Adapters — standard adapters may need to be replaced
- **MAC Address Spoofing**: Must be enabled on both OPNsense NICs (Advanced Features in VM Settings) since OPNsense routes traffic from other VMs
- **Virtual switch types matter**: External = reaches physical network; Internal = isolated lab network. LAN-side VMs must be on Internal to route through OPNsense
- **Disable Automatic Checkpoints**: Prevents silent config rollbacks on all lab VMs
- **Port 514 conflict**: rsyslog occupies UDP 514 on Ubuntu — use 5514 for Splunk syslog input
- **Subnet separation**: OPNsense LAN should use a different subnet than your home router (e.g., `10.0.0.0/24` if your home network is `192.168.1.0/24`)
- **OPNsense ISO**: Downloads as `.iso.bz2` — requires 7-Zip for extraction on Windows

---

## Recommended Splunk Add-ons

- **Splunk Add-on for Microsoft Windows** — CIM-compatible field extraction for Windows Event Logs
- **Splunk Add-on for Sysmon** — CIM-compatible knowledge for Sysmon telemetry
- **Splunk Add-on for pfSense/OPNsense** — Field extraction for firewall logs
- **Splunk Common Information Model (CIM)** — Data models for portable detections
- **Splunk Security Essentials (SSE)** — Pre-built security use cases and content

---

## Attack Simulation (Planned)

- **Atomic Red Team** — MITRE ATT&CK-aligned test cases for detection validation
- **Benign log generators** — Manual brute force (4625), privilege escalation (4672), encoded PowerShell commands (T1059.001)

---

## Quick Verification Searches

```spl
# All data sources at a glance
index=* earliest=-15m | stats count by index, sourcetype, host

# OPNsense firewall logs
index=firewall sourcetype=opnsense | head 10

# Windows Security events
index=windows sourcetype=WinEventLog:Security | head 10

# Sysmon process creation
index=sysmon EventCode=1 | table _time, Image, CommandLine, User, ParentImage

# Failed logons (brute force indicator)
index=windows EventCode=4625 | stats count by src_ip, TargetUserName

# PowerShell script block logging
index=windows EventCode=4104 | table _time, ScriptBlockText

# OPNsense traffic from a specific VM
index=firewall sourcetype=opnsense <WINDOWS_IP> | head 10
```

---

## Repository Structure

```
mini-soc-lab/
├── README.md
├── configs/
│   ├── splunk-enterprise/
│   │   └── inputs.conf
│   ├── splunk-forwarder/
│   │   ├── inputs.conf
│   │   └── outputs.conf
│   └── sysmon/
│       └── sysmonconfig-export.xml
├── detections/
│   ├── brute-force-detection.spl
│   ├── privilege-escalation.spl
│   └── suspicious-process.spl
└── docs/
    ├── architecture-diagram.png
    ├── build-guide.md
    └── troubleshooting.md
```

---

## Author

Built as a hands-on detection engineering lab to develop real-world SOC analyst skills including log analysis, detection writing, and incident triage.
