# Suricata Network IDS Integration: From Silent Capture to Correlated Alerts

## Objective
Deploy Suricata as a network-layer intrusion detection system (IDS) on the DVWA agent host, integrate its output into Wazuh, and validate detection using a live port scan. This extends Sentinel Forge's coverage beyond host-based detection (FIM, log analysis) into network-based visibility — catching reconnaissance and lateral-movement activity that never touches a file.

## Lab Environment
| Machine | OS | Purpose |
|---|---|---|
| Ubuntu 1 | Ubuntu 24.04  | Wazuh Manager |
| Ubuntu 2 | Ubuntu 26.04 | Wazuh Agent + Suricata sensor + DVWA |
| Kali Linux | Kali Rolling | Attacker machine (Nmap scans) |

All three VMs configured with **Bridged networking** in VirtualBox — required for Suricata to see real traffic between hosts; NAT mode would have hidden/translated packets in a way that undermines IDS visibility.

## Step 1 — Install Suricata
```bash
sudo apt install suricata -y
sudo suricata-update
```
Loaded the free Emerging Threats Open ruleset — 52,245 signatures, 0 failed to load.

## Step 2 — Configure the Monitored Interface
```bash
ip a
```
Identified the real active interface (`enp0s3`), then set it in Suricata's config:
```yaml
af-packet:
  - interface: enp0s3
```

## Step 3 — Wire Suricata's Output into Wazuh
On the agent's `ossec.conf`:
```xml
<localfile>
  <log_format>json</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

## Step 4 — Attack Simulation
From the dedicated **Kali Linux attacker VM**, against the Ubuntu 2 agent:
```bash
sudo nmap -sS -p- 192.168.29.250
```
`-sS` (SYN/stealth scan) requires root — an early attempt without `sudo` failed outright with "requires root privileges," a reminder to check the actual error output rather than assume a deeper problem.

## Results
**Named signature alert (no custom rule needed)** — Wazuh's built-in Suricata ruleset classified traffic automatically:
```
rule.id: 86601
rule.description: Suricata: Alert - SURICATA ICMPv4 invalid checksum
rule.level: 3
```

**Raw flow-level scan evidence** — even for scan techniques that didn't trip a named alert signature, Suricata's flow logging captured the unmistakable behavioral pattern of a SYN scan: dozens of connections from the Kali attacker's IP to the agent's IP across random ports within the same second, each showing `syn:true, rst:true, ack:true, state:closed` — the exact fingerprint of send-SYN/receive-RST/move-to-next-port scanning behavior.

## Troubleshooting

**Problem: `eve.json` stayed at 0 bytes despite confirmed real traffic**
Diagnosis: checked Suricata's actual interface configuration against the real system interface:
```bash
sudo grep -A 3 "af-packet:" /etc/suricata/suricata.yaml
ip route
```
**Root cause:** the interface name in `suricata.yaml` was `eth0s3` — a typo/mismatch against the real interface `enp0s3`. Suricata was listening on an interface that didn't exist, so it silently captured nothing despite the service reporting "active (running)" and the config validating successfully.

**Fix:** corrected the interface name to match `ip a`/`ip route` output exactly, restarted the service.

**Problem: `wazuh-agent.service` failed after adding the Suricata `<localfile>` block — "Extra content at the end of the document"**
Same class of bug as the manager's earlier syscheck incident: a duplicate `<ossec_config>` root element had been introduced, splitting the file into two separate XML documents.
```bash
sudo grep -n "ossec_config" /var/ossec/etc/ossec.conf
```
confirmed two opening and two closing tags where there should be exactly one of each.

**Fix:** removed the premature `</ossec_config>` and the following redundant `<ossec_config>`, merging everything into one continuous block.

**Problem: named "alert" events didn't appear for the Nmap scan itself**
The scan's SYN packets were correctly logged as `event_type: flow` data (proving Suricata saw every packet), but didn't trigger a named
