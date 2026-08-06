# Wazuh Agent Registration

## Objective

Register the Wazuh Agent running on **Ubuntu 2 (24.04 LTS)** with the Wazuh Manager hosted on **Ubuntu 1 (26.04)**. This enables the agent to securely forward endpoint telemetry, system logs, and security events to the Wazuh platform for centralized monitoring and analysis.

---

# Lab Environment

| Machine | OS | Purpose |
|---------|----|---------|
| Ubuntu 1 | Ubuntu 26.04 | Wazuh Manager + Wazuh Indexer + Wazuh Dashboard |
| Ubuntu 2 | Ubuntu 24.04 LTS | Wazuh Agent |

---

# Step 1 - Install the Wazuh Agent

Download the official Wazuh Agent installation script.

```bash
curl -so wazuh-agent.sh https://packages.wazuh.com/4.x/wazuh-agent.sh
```

Install the agent and register it with the Wazuh Manager.

```bash
sudo WAZUH_MANAGER="<Ubuntu1-IP>" bash wazuh-agent.sh -a
```

Replace `<Ubuntu1-IP>` with the IP address of your Ubuntu 1 Wazuh Manager.

---

# Step 2 - Enable and Start the Agent

Enable the Wazuh Agent service so it starts automatically after every reboot.

```bash
sudo systemctl enable wazuh-agent
```

Start the agent.

```bash
sudo systemctl start wazuh-agent
```

---

# Step 3 - Verify the Agent Status

Check whether the agent is running correctly.

```bash
sudo systemctl status wazuh-agent
```

Expected output:

```text
Active: active (running)
```

---

# Step 4 - Verify Registration from the Dashboard

Open the Wazuh Dashboard.

Navigate to:

```
Dashboard
→ Endpoints
```

Expected result:

- Agent Status: **Active**
- Agent Name: **linux22**
- Manager: **Ubuntu 1 (26.04)**

---

# Troubleshooting

## Agent Appears Offline

Restart the agent.

```bash
sudo systemctl restart wazuh-agent
```

Check the service logs.

```bash
sudo journalctl -u wazuh-agent -f
```

---

## Verify Network Connectivity

Ensure the agent can communicate with the manager.

```bash
ping <Ubuntu1-IP>
```

Verify the manager IP configured in the agent.

```bash
grep address /var/ossec/etc/ossec.conf
```

---

# Verification Checklist

- [x] Wazuh Agent installed successfully
- [x] Agent registered with Ubuntu 1
- [x] Agent service running
- [x] Agent displayed as **Active** in the Wazuh Dashboard
- [x] Secure communication established between Agent and Manager

---

# Screenshots
![Uploading wazuhAgent_setup.jpg…]()



Include the following:

- Wazuh Agent installation terminal
- `systemctl status wazuh-agent`
- Wazuh Dashboard showing the agent status as **Active**
