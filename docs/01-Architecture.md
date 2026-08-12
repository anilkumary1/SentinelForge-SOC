# SentinelForge-SOC Architecture

## Project Overview

SentinelForge-SOC is an enterprise-style SOC homelab built to simulate real-world attack detection, monitoring, and incident response.

The project uses Wazuh as the SIEM/XDR platform with custom detection engineering, Suricata for network IDS, VirusTotal for malware reputation, TheHive for case management, and DVWA for attack simulation.

---

# Lab Environment

## Virtual Machines

### Ubuntu 1 (SOC Server)

Services running:

- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard

Purpose:

- Receives logs from agents
- Correlates events
- Applies custom detection rules
- Stores alerts
- Visualizes security events

---

### Ubuntu 2 (Endpoint)

Services running:

- Wazuh Agent
- DVWA
- Apache Web Server

Purpose:

- Generates attack telemetry
- Sends logs to Wazuh Manager
- Simulates a monitored production server

---

# Data Flow

```
                Attacker
                    │
                    ▼
              DVWA Web Server
             (Ubuntu Agent)
                    │
        Apache Access Logs
                    │
                    ▼
             Wazuh Agent
                    │
           Secure Communication
                    │
                    ▼
             Wazuh Manager
                    │
          Rule Correlation Engine
                    │
                    ▼
             Wazuh Indexer
                    │
                    ▼
            Wazuh Dashboard
```

---

# Components

| Component | Purpose |
|-----------|----------|
| Wazuh Manager | Event analysis and rule engine |
| Wazuh Indexer | Stores alerts |
| Wazuh Dashboard | SOC dashboard |
| Wazuh Agent | Collects endpoint logs |
| Apache | Generates web access logs |
| DVWA | Attack simulation platform |
| Custom Rules | Detect XSS, Brute Force and other attacks |

---

# Detection Workflow

1. User launches an attack against DVWA.
2. Apache writes the request into access.log.
3. Wazuh Agent monitors the log.
4. Log is sent to Wazuh Manager.
5. Built-in rules analyze the event.
6. Custom SentinelForge rules increase severity.
7. Alert is indexed.
8. Analyst investigates in Wazuh Dashboard.

---

# Current Progress

✅ Wazuh Installed

✅ Agent Connected

✅ Apache Log Monitoring

✅ DVWA Integrated

✅ XSS Detection

✅ SSH Brute-force Detection

✅ XSS Detection

✅ SSH Brute-force Detection
