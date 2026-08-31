# 🔑 Credential Harvesting, Pass-the-Hash & Lateral Movement

## 🎯 1. Overview & Attack Progression

Once local administrative privileges (`NT AUTHORITY\SYSTEM` or local `Administrator`) are obtained on a foothold system, the adversary shifts focus to **credential harvesting** and **lateral movement** to compromise high-privilege domain identities (e.g., Domain Admins or high-tier Service Accounts).

``` text
+---------------------+        Dump LSASS / SAM        +-------------------------+
| Local Administrator |  --------------------------->  | Harvested NTLM / Kerb   |
|   (Foothold Host)   |                                | Credentials of Logged-In|
+---------------------+                                | Domain Accounts         |
                                                       +-------------------------+
                                                                    |
                                                                    | Pass-the-Hash (sekurlsa::pth)
                                                                    v
+---------------------+     WMI / SMB Reconnaissance   +-------------------------+
|   Domain / Forest   |  <---------------------------  | Elevated Domain Session |
|  Admin Compromise   |                                | (e.g., emp-svc context) |
+---------------------+                                +-------------------------+
```

---

## 🧠 2. Core Concepts & Mechanics

### Why Target Service Accounts & Logged-on Sessions?

* **High Privileges:** Service accounts often hold elevated privileges across multiple hosts or servers in the Active Directory domain to manage services, backups, or database synchronization.
* **Token & Secret Residuals:** When a domain user or service account authenticates to a workstation or server, Windows caches authentication material (NTLM hashes, Kerberos tickets, and occasionally plaintext credentials) inside the memory space of the **Local Security Authority Subsystem Service (LSASS)** process.
* **Session Hunting:** Finding where high-value users have active sessions allows an attacker to compromise those specific machines and harvest their credentials.

---

## 🔍 3. Active Session Hunting (`Invoke-UserHunter`)

Before dumping credentials blindly, attackers identify machines where Domain Admins or targeted service accounts maintain active logon sessions.

```powershell
# Load PowerView
. .\PowerView.ps1

# Locate all active sessions of Domain Admins across domain computers
Invoke-UserHunter -Verbose

# Target specific user groups or accounts
Invoke-UserHunter -GroupName "Domain Admins" -Verbose

```

* **Mechanism:** Queries the Domain Controller for computers via LDAP and checks remote registry / NetWkstaUserEnum API calls to identify logged-on users.

---

## 💀 4. Credential Dumping via `Invoke-Mimikatz`

`Invoke-Mimikatz` leverages reflective DLL injection to load the Mimikatz binary entirely in memory without writing the executable to disk, helping bypass basic signature-based file detection.

### Dumping In-Memory Credentials (LSASS)

*Requires elevated local administrative privileges (`SeDebugPrivilege`).*

```powershell
# Import script into the current PowerShell session
. .\Invoke-Mimikatz.ps1

# Dump SAM hashes, LSA secrets, and LSASS memory credentials
Invoke-Mimikatz -DumpCreds -Verbose

```

### What Gets Extracted?

* **NTLM Hashes:** Useful for Pass-the-Hash (PtH) attacks.
* **Kerberos Tickets:** TGTs and TGS tickets stored in memory (used for Pass-the-Ticket / Silver / Golden Ticket attacks).
* **Cleartext Passwords:** Extracted via `wdigest` or `tspkg` on unpatched/legacy systems.

---

## 🔄 5. Pass-the-Hash (PtH) Attack [T1550.002]

### The Concept

The **Pass-the-Hash (PtH)** technique allows an attacker to authenticate to remote systems or Active Directory services using the underlying **NTLM hash** (or RC4 Kerberos key) without ever needing or cracking the cleartext password.

### Execution via Mimikatz (`sekurlsa::pth`)

Using the dumped NTLM hash of a high-privilege domain account (e.g., `emp-svc`):

```powershell
Invoke-Mimikatz -Command ' "sekurlsa::pth /user:emp-svc /domain:cyberwarfare.corp /rc4:<NTLM_HASH> /run:powershell.exe" ' -Verbose

```

### Breakdown of Parameters

* **/user:** Target domain account username.
* **/domain:** Fully Qualified Domain Name (FQDN) or NetBIOS domain name.
* **/rc4:** The target account's NTLM hash (represented as an RC4 key).
* **/run:** The process to spawn with the injected identity token (typically `powershell.exe` or `cmd.exe`).

> **How it Works:** Mimikatz creates a new process under your current local context with a dummy Kerberos/NTLM credential token. When this spawned PowerShell process makes an outbound network connection (via SMB, RPC, or WMI), Windows automatically uses the injected NTLM hash to authenticate against the remote target.

---

## 🔎 6. Identifying Admin Access via WMI (`Find-WMILocalAdminAccess`)

Once inside the new PowerShell session spawned by the PtH attack, verify where the compromised identity possesses local administrative rights.

```powershell
# Import PowerView / LocalAdminAccess script
. .\Find-WMILocalAdminAccess.ps1

# Scan domain machines to find where current credentials have Local Admin access over WMI
Find-WMILocalAdminAccess -Verbose

```

* **Mechanism:** Queries domain computers and attempts an authenticated WMI connection (`win32_computersystem`).
* **Result:** Returns a list of hostnames/IPs where the currently elevated domain identity (`emp-svc`) can execute remote code, deploy implants, or continue lateral movement.


---
## 7. Dot Sourcing
    The difference lies in how PowerShell executes the script: whether it runs in a **new isolated child scope** or directly in your **current session scope**.

* **`.\Invoke-Mimikatz.ps1`**: Runs the script in a **child scope**. Variables, functions, and modules defined inside the script are created temporarily and disappear as soon as the script finishes executing.
* **`. .\Invoke-Mimikatz.ps1`** (with a dot and a space before the path, known as **dot-sourcing**): Runs the script in the **current scope**. Any functions (such as `Invoke-Mimikatz`) or variables defined inside the script are loaded directly into your active PowerShell session so you can actually call and use them.

Using dot-sourcing is required when loading standalone PowerShell tool scripts like Mimikatz, PowerView, or PowerUp, because running them normally without the dot will execute any auto-run blocks if present, but will fail to expose the underlying command functions for you to use.

---
## 🛡️ 8. Detection & Defensive Mitigations

| Threat Vector | Defensive Mitigation |
| --- | --- |
| **LSASS Dumping** | Enable **LSA Protection (RunAsPPL)** and **Credential Guard** (Virtualization-based Security) to block non-protected processes from reading LSASS memory. |
| **Pass-the-Hash** | Implement **Tiered Administration** models and place administrative accounts into the **Protected Users** security group (which disables NTLM caching and delegation). |
| **Over-Privileged Service Accounts** | Use **Group Managed Service Accounts (gMSA)** with automatic password rotation and restrict service accounts from logging on interactively (`Deny log on locally`). |
| **Lateral Movement Monitoring** | Audit Windows Event ID `4624` (Logon Type 3 - Network Logon using NTLM authentication) and Sysmon Event ID `10` (Process access to `lsass.exe`). |

