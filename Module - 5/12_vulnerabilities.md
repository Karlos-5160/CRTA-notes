Almost none of the techniques you executed rely on software bugs, unpatched code flaws, or zero-day exploits in Windows Server 2012 or 2016.

Instead, they exploit **architectural protocol behaviors (features working as designed)** combined with **administrative misconfigurations and poor operational security (OpSec)**.

---

### Breakdown: Flaw vs. Protocol Behavior vs. Misconfiguration

| Technique Executed | Is it an OS Bug / CVE? | What Actually Made It Possible? |
| --- | --- | --- |
| **Service `binpath` Escalation** | **No** | **Administrative Misconfiguration:** Someone or a third-party installer set a weak Discretionary Access Control List (DACL) on the service, granting standard users write/modify permissions (`SERVICE_CHANGE_CONFIG`).

 |
| **Credential Dumping (SAM / LSASS)** | **No** | **Expected OS Architecture:** Windows must store hashes and keys in memory/disk to handle active authentication sessions. Dumping them only succeeded because you had *already* escalated to local Administrator/SYSTEM.

 |
| **Pass-the-Hash (PtH)** | **No** | **Protocol Design:** The NTLM authentication protocol relies on the hash itself as the secret proof of identity. The protocol has no native mechanism to distinguish between the hash and the cleartext password. |
| **Kerberoasting** | **No** | **Kerberos Protocol Design + Weak Passwords:** Kerberos allows *any* valid domain user to request a service ticket (TGS) for any Service Principal Name (SPN). The flaw was human error: assigning an SPN to a standard user account with a weak, crackable password.

 |
| **PowerShell Remoting (WinRM)** | **No** | **Built-in Administrative Feature:** WinRM is Microsoft’s intended remote administration tool, enabled by default on Server 2012+. You simply authenticated to it using legitimate harvested administrative credentials.

 |
| **Golden & Silver Tickets** | **No** | **Cryptographic Design (Feature by Design):** Kerberos relies on symmetric encryption. If the key holder signs a ticket, systems must trust it. The compromise happened because you stole the master key (`krbtgt` or the computer account hash via DCSync).

 |

---

### Why Attackers Focus on These Vectors in Modern AD

1. **Patch-Proof:** Because these are foundational features of Active Directory (Kerberos, NTLM, SCM, RPC, WinRM), applying standard Windows security updates does not stop them.


2. **Blending In:** Exploiting administrative features generates legitimate traffic (e.g., standard Kerberos ticket requests or WinRM management calls), making detection significantly harder than signature-based malware.


3. **The Common Denominator:** In an Active Directory environment, privilege escalation almost always stems from:
* Over-privileged service accounts (e.g., service accounts placed in `Domain Admins`).
* Password reuse across local administrator accounts.
* Weak, human-chosen passwords assigned to accounts with SPNs.


* Insecure object or service permissions assigned during third-party software installations.