# Xray VLESS + REALITY Deployment on Azure

![Azure](https://img.shields.io/badge/Cloud-Microsoft%20Azure-0078D4?logo=microsoftazure&logoColor=white)
![Linux](https://img.shields.io/badge/OS-Linux-FCC624?logo=linux&logoColor=black)
![Xray](https://img.shields.io/badge/Network-Xray-5E81AC)
![VLESS](https://img.shields.io/badge/Protocol-VLESS-6C5CE7)
![REALITY](https://img.shields.io/badge/Security-REALITY-2E8B57)

A personal deployment guide for installing and configuring [Xray-core](https://github.com/XTLS/Xray-core) on a Linux virtual machine using **VLESS + REALITY + Vision**.

This README documents the server setup, key generation, firewall configuration, client configuration, and connectivity testing.

> **Environment:** Azure Linux VM, systemd, UFW, Xray-core, Windows, and v2rayN.

## Table of Contents

- [Architecture](#architecture)
- [1. Configure the Firewall](#1-configure-the-firewall)
- [2. Install Xray](#2-install-xray)
- [3. Generate Credentials](#3-generate-credentials)
- [4. Configure the Server](#4-configure-the-server)
- [5. Validate and Start Xray](#5-validate-and-start-xray)
- [6. Test Server Connectivity](#6-test-server-connectivity)
- [7. Configure the Client](#7-configure-the-client)
- [8. Test the Connection](#8-test-the-connection)
- [9. Troubleshooting](#9-troubleshooting)
- [10. Security Notes](#10-security-notes)

## Architecture

The deployment uses a Linux VM as the Xray server. A compatible client connects to the server using VLESS over TCP with REALITY security and Vision flow.

```text
Windows Client
    |
    | VLESS + REALITY + Vision
    | TCP port 443
    v
Azure VM
    |
    | Azure Network Security Group
    | UFW firewall
    v
Xray-core
    |
    | Freedom outbound
    v
Destination Services
```

## 1. Configure the Firewall

Allow inbound TCP traffic on port `443`.

On the Linux VM:

```bash
sudo ufw allow 443/tcp
sudo ufw status
```

If UFW is not enabled and you intend to use it, review the existing firewall rules and ensure SSH access is allowed before enabling it.

In Azure, also verify that the VM's Network Security Group allows inbound TCP traffic on port `443`.

## 2. Install Xray

Install Xray using the official XTLS installation script.

```bash
sudo bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

The installer normally places the configuration file at:

```text
/usr/local/etc/xray/config.json
```

Check the installed version:

```bash
xray version
```

Review the official installation instructions before running installation scripts with elevated privileges.

## 3. Generate Credentials

Generate the credentials needed for the server and client.

### Generate a UUID

```bash
xray uuid
```

Save the generated UUID as `YOUR_UUID`.

### Generate REALITY keys

```bash
xray x25519
```

Record the generated private and public keys.

- `YOUR_PRIVATE_KEY` is used on the server.
- `YOUR_PUBLIC_KEY` is used by the client.

Use the actual field names printed by your installed Xray version.

### Generate a short ID

```bash
openssl rand -hex 8
```

Save the result as `YOUR_SHORT_ID`. Use the same short ID on both the server and client.

**Keep the private key and client credentials secret.** Never commit actual credentials to a public GitHub repository.

## 4. Configure the Server

Open the server configuration:

```bash
sudo nano /usr/local/etc/xray/config.json
```

Use the following configuration as a starting point. Replace all placeholders with your generated credentials.

```json
{
  "inbounds": [
    {
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "YOUR_UUID",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "dest": "www.zoom.us:443",
          "serverNames": [
            "www.zoom.us"
          ],
          "privateKey": "YOUR_PRIVATE_KEY",
          "shortIds": [
            "YOUR_SHORT_ID"
          ]
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom"
    }
  ]
}
```

### Configuration overview

| Setting | Purpose |
|---|---|
| `port` | Port on which Xray accepts connections |
| `protocol` | VLESS inbound protocol |
| `id` | Client UUID |
| `flow` | Vision flow setting |
| `security` | Enables REALITY transport security |
| `dest` | REALITY destination used for the handshake |
| `serverNames` | Accepted server name for the REALITY handshake |
| `privateKey` | Server-side REALITY private key |
| `shortIds` | Permitted client short IDs |
| `freedom` | Allows outbound connections to destinations |

The example uses `www.zoom.us` as the REALITY destination. Confirm that your selected destination is suitable for your environment and compatible with your Xray version.

## 5. Validate and Start Xray

Test the configuration before restarting the service:

```bash
sudo xray run -test -config /usr/local/etc/xray/config.json
```

If validation succeeds, restart and enable the service:

```bash
sudo systemctl restart xray
sudo systemctl enable --now xray
```

Check the service status:

```bash
sudo systemctl status xray
```

Inspect recent logs:

```bash
sudo journalctl -u xray -n 50 --no-pager
```

Follow logs in real time when troubleshooting:

```bash
sudo journalctl -u xray -f
```

### Confirm that port 443 is listening

```bash
sudo ss -tlnp | grep ':443'
```

A listening socket indicates that a process has bound to the port. It does not, by itself, prove that a remote client can connect successfully.

## 6. Test Server Connectivity

From Windows PowerShell, replace `YOUR_VPS_IP` with the VM's public IP address.

```powershell
Test-NetConnection YOUR_VPS_IP -Port 443
```

Look for:

```text
TcpTestSucceeded : True
```

This checks TCP connectivity only. It does not verify the VLESS credentials or the REALITY handshake.

## 7. Configure the Client

Use v2rayN or another compatible Xray client.

Create a VLESS profile with the following values:

| Field | Value |
|---|---|
| Address | `YOUR_VPS_IP` |
| Port | `443` |
| UUID | `YOUR_UUID` |
| Encryption | `none` |
| Flow | `xtls-rprx-vision` |
| Network | `tcp` |
| Security | `reality` |
| Server name (SNI) | `www.zoom.us` |
| Fingerprint | `chrome` |
| Public key | `YOUR_PUBLIC_KEY` |
| Short ID | `YOUR_SHORT_ID` |

Ensure that the REALITY public key, UUID, short ID, server name, and flow match the server configuration.

### Example client core configuration

The following JSON illustrates a local SOCKS proxy and a VLESS outbound. It is a core configuration example, not necessarily a directly importable v2rayN profile.

```json
{
  "inbounds": [
    {
      "listen": "127.0.0.1",
      "port": 10808,
      "protocol": "socks",
      "settings": {
        "udp": true
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "vless",
      "settings": {
        "vnext": [
          {
            "address": "YOUR_VPS_IP",
            "port": 443,
            "users": [
              {
                "id": "YOUR_UUID",
                "encryption": "none",
                "flow": "xtls-rprx-vision"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "serverName": "www.zoom.us",
          "fingerprint": "chrome",
          "publicKey": "YOUR_PUBLIC_KEY",
          "shortId": "YOUR_SHORT_ID"
        }
      }
    }
  ]
}
```

The SOCKS listener is bound to `127.0.0.1`, so it is accessible only from the local machine by default.

## 8. Test the Connection

Start the client and connect to the configured profile.

If the local SOCKS listener is running on port `10808`, test an HTTPS request from Windows using a terminal with `curl`:

```bash
curl -I -x socks5h://127.0.0.1:10808 https://facebook.com
```

The `socks5h` scheme requests hostname resolution through the SOCKS proxy.

A successful HTTP response indicates that the request passed through the proxy. The exact HTTP status can vary by destination and does not need to be `200`.

## 9. Troubleshooting

| Symptom | What to investigate |
|---|---|
| Xray fails to start | Validate the JSON and inspect `journalctl` |
| Port 443 is not listening | Check the service status, configuration, and port conflicts |
| TCP test fails | Check the VM public IP, Azure NSG, UFW, and routing |
| TCP works but the client fails | Check UUID, public key, short ID, server name, and flow |
| Client connects but requests fail | Inspect client/server logs, outbound connectivity, and DNS behavior |
| Port 443 is already in use | Identify the process with `sudo ss -tlnp` |

A useful troubleshooting sequence is:

1. Validate the server configuration.
2. Check whether the Xray service is running.
3. Confirm that port 443 is listening.
4. Test TCP reachability from the client.
5. Inspect the client and server logs during a connection attempt.
6. Verify that client and server settings match.


## Learning Outcomes

This project provides hands-on practice with:

- Provisioning and accessing a Linux VM on Azure.
- Installing and managing software on Linux.
- Managing systemd services and reading logs.
- Configuring host and cloud firewalls.
- Exploring TCP connectivity and client-server communication.
- Configuring a client and troubleshooting connectivity issues.

## References

- [Xray-core](https://github.com/XTLS/Xray-core)
- [Xray installation script](https://github.com/XTLS/Xray-install)
- [Azure Virtual Machines documentation](https://learn.microsoft.com/azure/virtual-machines/)
- [v2rayN](https://github.com/2dust/v2rayN)

---

*This is a personal learning and deployment reference. Validate all configuration values against the installed Xray version and your own environment before using them.*
