# OPNsense Firewall Rules — Lab Configuration Notes

## Interface Assignments

| Role    | Hyper-V vSwitch | OPNsense Interface |
|---------|-----------------|--------------------|
| WAN     | External-WAN    | hn0 (first NIC)    |
| LAN     | Internal-LAN    | hn1 (second NIC)   |

## Default LAN IP
- OPNsense LAN gateway: `192.168.1.1`
- Web GUI: `http://192.168.1.1`
- Default credentials: `root` / `opnsense` (change immediately)

## LAN DHCP Range
- Enabled by default
- Range: `192.168.1.100` – `192.168.1.200`
- All lab VMs (Windows Server, Ubuntu) receive IPs from here

## Firewall Rules Applied

### LAN → Any (for lab traffic)
- Action: Pass
- Interface: LAN
- Protocol: Any
- Source: LAN net
- Destination: Any
- Purpose: Allow all internal lab VM traffic outbound

## Logging Configuration (for Splunk Integration)
- Enable: `System → Logging → Settings → Enable remote logging`
- Remote syslog server: `<ubuntu-splunk-ip>`
- Port: `514` (UDP)
- Facility: Any

> Note: OPNsense syslog ingestion into Splunk is a planned enhancement
> to enable C2 detection, beaconing patterns, and data exfiltration use cases.
> Add a [monitor:///var/log/opnsense/] or UDP 514 input stanza in Splunk
> inputs.conf once syslog forwarding is verified.
