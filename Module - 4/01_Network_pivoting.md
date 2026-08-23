# 🔀 Network Pivoting, Dynamic Port Forwarding & Proxychains

## 📖 1. Core Fundamental Concepts

### What is Network Pivoting?
**Network Pivoting** is a post-exploitation technique where an attacker uses a compromised machine (the "foothold" or "pivot host") to route traffic and launch attacks against other isolated, internal networks that are otherwise unreachable from the outside.

### What is a SOCKS Proxy?
A **SOCKS (Socket Secure) Proxy** is a Layer-4/Layer-5 protocol that transparently relays arbitrary TCP (and optionally UDP) client-server traffic through an intermediary proxy server. 
* Unlike standard HTTP proxies (which only understand web traffic), a SOCKS proxy does not interpret or alter application-layer data.
* It simply acts as a blind pipe, establishing a connection to the requested IP and port on behalf of the client and streaming raw data back and forth.

### What is Proxychains?
**Proxychains** is a Linux command-line utility that forces any arbitrary TCP client (like `nmap`, `curl`, `nc`, `rdesktop`, `impacket`) to redirect its connections through one or more SOCKS4, SOCKS5, or HTTP proxies.
* **Mechanism:** It uses `LD_PRELOAD` to dynamically hook into and hijack standard C library network system calls (specifically `connect()`).
* **Why it matters:** It allows offensive tools that have no built-in proxy support to tunnel their traffic across internal target networks.

---

## 📐 2. Architecture Overview & Pivot Scenario

In enterprise networks and Red Team engagements, high-value identity infrastructure (Domain Controllers, database servers, employee workstations) resides within an isolated internal subnet (`10.10.10.0/24`) with no direct routing to or from the external attacker (`192.168.50.0/24`)[cite: 6].

```text
+---------------------------------------------------------------------------------------------------+
|                                      NETWORK PIVOT TOPOLOGY                                       |
+---------------------------------------------------------------------------------------------------+

       [ ATTACKER MACHINE ]                  [ DMZ PIVOT HOST ]                      [ INTERNAL AD NETWORK ]
  (Host-Only Subnet: 192.168.50.0/24)                                      (Isolated Subnet: 10.10.10.0/24)

  +--------------------------+             +--------------------------+             +--------------------------+
  |        Kali Linux        |             |   Web-Server (Dual-NIC)  |             |     Domain Controller    |
  |      [Attacker VM]       |             |     [Metasploitable]     |             |       [Target VM]        |
  |                          |             |                          |             |                          |
  |  Adapter 3 (eth0)        | <---------> |  Adapter 3 (eth0)        |             |  Adapter 4 (eth0)        |
  |  IP: 192.168.50.2/24     |  (SSH / DMZ)|  IP: 192.168.50.3/24     |             |  IP: 10.10.10.2/24       |
  |  GW: 192.168.50.1        |             |  GW: 192.168.50.1        |             |  GW: 10.10.10.1          |
  +--------------------------+             +------------+-------------+             +--------------------------+
                                                        |                                        ^
                                                        |  Adapter 4 (eth1)                      |
                                                        |  IP: 10.10.10.5/24 (Host-Only #4)      |
                                                        +----------------------------------------+
```

---

## 🧠 3. Why Pivoting is Necessary (The Tooling Problem)

### ❓ Why Not Just Use a Standard SSH Shell?

When an attacker compromises a perimeter dual-homed server and obtains an interactive SSH shell:
* **Tooling Absence:** The compromised host is just a server; it does not contain penetration testing toolsets (Impacket, BloodHound/SharpHound, CrackMapExec/NetExec, Responder, wordlists)[cite: 6].
* **No GUI Capabilities:** Graphical clients (like `rdesktop` / `xfreerdp`) cannot be displayed or run natively inside a raw terminal shell on the pivot host[cite: 6].
* **OpSec & Artifact Footprint:** Uploading large toolkits, Python runtimes, and binaries onto the target increases detection risk[cite: 6].

**Pivoting solves this** by allowing the attacker to run all tools locally from their Kali machine while transparently relaying the network packets through the pivot host[cite: 6].

> **Pivoting** allows the attacker to keep all tools, exploits, and wordlists locally on the Kali machine while transparently tunneling Layer-4/Layer-7 traffic through the compromised host into the internal subnet.
---

## ⚙️ 4. Configuring Proxychains (`/etc/proxychains4.conf`)

Before starting the tunnel, prepare the Proxychains configuration on the attacker (Kali) machine:

```bash
sudo nano /etc/proxychains4.conf
```

1. **Comment out `proxy_dns`:**
   ```text
   # proxy_dns
   ```
   *(Crucial: Disabling `proxy_dns` prevents tool hangs and timeouts when connecting to direct internal IP addresses)[cite: 6].*

2. **Define the Proxy Server at the Bottom:**
   Ensure the `[ProxyList]` section points to your planned local port (e.g., `8090`)[cite: 6]:
   ```text
   [ProxyList]
   socks5 127.0.0.1 8090
   ```

---

## 🔀 5. Dynamic Port Forwarding (SSH SOCKS Tunnel)

Dynamic port forwarding instructs OpenSSH to spin up a local SOCKS proxy on your Kali machine and relay requested TCP connections through the remote SSH server (`sshd`)[cite: 6].

### Step-by-Step Implementation

#### Step 1: Handling Legacy SSH Host Keys
Older Linux targets (like Metasploitable 2) offer older key types (`ssh-rsa`, `ssh-dss`) that modern OpenSSH disables[cite: 6].

*Append to `~/.ssh/config` on Kali:*
```text
Host 192.168.50.*
    HostKeyAlgorithms +ssh-rsa,ssh-dss
    PubkeyAcceptedKeyTypes +ssh-rsa
```

#### Step 2: Establish the Dynamic SOCKS Proxy
Run the SSH dynamic port forwarding command:

```bash
# Option A: Interactive Foreground Session
ssh -D 8090 msfadmin@192.168.50.3

# Option B: Background Persistent Session (No remote shell, fork to background)
ssh -f -N -D 8090 msfadmin@192.168.50.3
```

* **`-D 8090`**: Binds a SOCKS proxy to local port `8090`[cite: 6].
* **`-f`**: Requests SSH to go to the background just before command execution[cite: 6].
* **`-N`**: Do not execute a remote command/shell (tunnel only)[cite: 6].

#### Step 3: Verify the Local SOCKS Listener
```bash
netstat -ant | grep 8090
# OR
ss -tulpn | grep 8090
```
*(You should see `127.0.0.1:8090` in `LISTEN` state)[cite: 6].*


### Summary 
1. **Local SOCKS Listener:** OpenSSH spawns a local **SOCKS4/SOCKS5 proxy** listening on `127.0.0.1:8090` on the Kali machine.
2. **Encapsulated Tunnel:** Traffic directed to `localhost:8090` is encapsulated within the encrypted SSH transport layer and transmitted to `sshd` on the pivot host (`192.168.50.3`)[cite: 3].
3. **Remote Egress Routing:** The pivot host's `sshd` unpacks the SOCKS request and asks its local OS kernel to connect to the internal target (e.g., `10.10.10.2:445`)[cite: 3].
4. **Kernel Interface Selection:** The pivot host's kernel evaluates its own routing table:
   * It identifies that `10.10.10.2` falls within `10.10.10.0/24` (attached to `eth1`)[cite: 3].
   * It sends an ARP request on `eth1`, establishes the TCP handshake with the target, and streams the data back through the SSH tunnel.

---

## 🎯 6. Interacting with Internal Targets

With Proxychains configured and the SOCKS tunnel active, prefix any command with `proxychains` to attack internal targets[cite: 6]:

```bash
# 1. Verify TCP Port & Service Reachability
proxychains nc -nv 10.10.10.2 445

# 2. Port Scanning (Must use TCP Connect scan -sT and disable ping -Pn)
proxychains nmap -sT -Pn -p 88,135,139,445,3389 10.10.10.2

# 3. GUI Remote Desktop (RDP) Access into Internal Workstation
proxychains rdesktop 10.10.10.4
# OR
proxychains xfreerdp /v:10.10.10.4 /u:karlos /p:windows7

# 4. Active Directory Enumeration via Impacket / NetExec
proxychains crackmapexec smb 10.10.10.2 -u "dc_user" -p "Password123!"
```

---

## ⚠️ 7. Critical Technical Constraints & Best Practices

| Constraint | Technical Reason | Solution / Workaround |
| :--- | :--- | :--- |
| **No ICMP (`ping`) via SOCKS** | SOCKS4/SOCKS5 works at Layer 4/5 (TCP/UDP); it cannot encapsulate Layer 3 ICMP packets[cite: 6]. | Use `proxychains nc -nv <IP> <Port>` or `proxychains nmap -sT -Pn` to verify reachability[cite: 6]. |
| **No SYN Stealth Scans (`nmap -sS`)** | SYN scans use raw Layer-3 sockets, which bypass `LD_PRELOAD` hooks[cite: 6]. | Always specify `-sT` (TCP Connect) and `-Pn` (No Ping) with `nmap`[cite: 6]. |
| **Session Persistence** | The SOCKS proxy terminates instantly if the underlying SSH process drops[cite: 6]. | Use `ssh -f -N -D 8090` or keep the session inside `tmux`/`screen`[cite: 6]. |