# Lab Environment

| Machine | OS | Purpose |
|---------|----|---------|
| Ubuntu 1 | Ubuntu 24.04 LTS | Wazuh Manager + Wazuh Indexer + Wazuh Dashboard |
| Ubuntu 2 | Ubuntu 24.04 LTS | Wazuh Agent + Apache + Docker + DVWA |

## Step 1 - Install Wazuh Server (Ubuntu 1)

Download the installation assistant.

```bash
curl -sO https://packages.wazuh.com/4.13/wazuh-install.sh
```

Run the installer.

```bash
sudo bash wazuh-install.sh -a
```

This installs:

- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard

Verification

```bash
sudo systemctl status wazuh-manager

sudo systemctl status wazuh-indexer

sudo systemctl status wazuh-dashboard
```

Check the IP address.

```bash
ip a
```

Access Dashboard

```
https://<Ubuntu1-IP>
```

<img width="1637" height="961" alt="wazuh-setup" src="https://github.com/user-attachments/assets/a30fd047-b556-4fc8-bf4b-224ca7f1b637" />


- Installation completed
- Dashboard credentials generated
