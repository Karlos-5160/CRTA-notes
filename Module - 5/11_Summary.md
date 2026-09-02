# 🗺️ End-to-End Active Directory Attack Lifecycle & Architecture 

## 🧭 1. Complete Attack Progression Flowchart

The following diagram illustrates the structured progression through the internal kill chain, transitioning from initial local foothold on a member workstation to enterprise persistence across the Active Directory forest.

```text
+---------------------------------------------------------------------------------------------------+
|                                  ACTIVE DIRECTORY ATTACK PATHWAY                                  |
+---------------------------------------------------------------------------------------------------+

   [ PHASE 1: INITIAL ACCESS & NETWORK PIVOT ]
   • Attacker accesses perimeter DMZ host (e.g., dual-homed Linux server).
   • Establishes SSH Dynamic Port Forwarding (SOCKS5 proxy on port 8090/9050).
   • Configures Proxychains on Kali to route Layer 4/7 tools into internal subnet (10.10.10.0/24).
                                      |
                                      v
   [ PHASE 2: LOCAL PRIVILEGE ESCALATION (WORKSTATION) ]
   • Gained initial execution as low-privilege domain user (CYBERWARFARE\employee).
   • Identified weak DACL on local service via automated/manual checks (PowerUp.ps1).
   • Abused Service Control Manager: modified service binpath (sc.exe config snmptrap).
   • Executed payload under LocalSystem context to elevate to local BUILTIN\Administrators.
                                      |
                                      v
   [ PHASE 3: CREDENTIAL HARVESTING & SESSION HUNTING ]
   • Performed active session enumeration (Invoke-UserHunter) to locate Tier-1/Tier-0 targets.
   • Exported local registry hives (HKLM\SAM and HKLM\SYSTEM) to recover local hashes.
   • Injected into LSASS memory via Invoke-Mimikatz (-DumpCreds) to extract cached NTLM 
     hashes for domain service accounts (app-svc / emp-svc).
                                      |
                                      v
   [ PHASE 4: PASS-THE-HASH & LATERAL MOVEMENT ]
   • Spawned elevated execution context using harvested NTLM hash (sekurlsa::pth).
   • Verified remote administrative privileges over RPC/WMI (Invoke-CheckLocalAdminAccess).
   • Established WinRM remoting session (New-PSSession / Enter-PSSession over TCP 5985)
     to execute commands directly on internal member servers and Domain Controllers.
                                      |
                                      v
   [ PHASE 5: DOMAIN PRIVILEGE ESCALATION (KERBEROASTING) ]
   • Queried Active Directory LDAP for user accounts with registered SPNs (Get-NetUser -SPN).
   • Requested TGS tickets for target service accounts (Request-SPNTicket).
   • Exported ticket artifacts (.kirbi) from local session memory (kerberos::list /export).
   • Performed offline brute-force cracking to extract cleartext service credentials.
                                      |
                                      v
   [ PHASE 6: DOMAIN PERSISTENCE (FORGED TICKETS) ]
   • Leveraged Domain Admin privileges on DC to DCSync the KRBTGT account secret key.
   • Forged a Ticket Granting Ticket (Golden Ticket) valid for extended periods.
   • Injected TGT into session memory cache (/ptt) to maintain persistent, administrative
     access across the domain regardless of standard user password resets.

```

---

## 🔍 2. Detailed Phase Breakdown

### Phase 1: Perimeter Access & Pivoting

* **Objective:** Route attack tooling into an isolated internal subnet.
* **Mechanism:** OpenSSH dynamic forwarding (`-D`) binds a local SOCKS listener on Kali. The pivot host's kernel consults its local routing table to forward encapsulated traffic across its secondary internal interface (`eth1`).
* **Tooling:** OpenSSH, Proxychains (`/etc/proxychains4.conf`).

### Phase 2: Local Privilege Escalation

* **Objective:** Elevate from an unprivileged domain user to local administrative or `SYSTEM` authority on the local workstation.
* **Mechanism:** Exploitation of weak Discretionary Access Control Lists (DACLs) on Windows Services. Overwriting the service binary path (`binpath`) directs the Service Control Manager to execute administrative commands when the service restarts.
* **Tooling:** `PowerUp.ps1`, `sc.exe`, `net localgroup`.

### Phase 3: Credential Harvesting

* **Objective:** Obtain credentials of higher-privileged domain entities logged into the local machine.
* **Mechanism:** Extracting authentication material from on-disk hives (`HKLM\SAM` decrypted via `HKLM\SYSTEM`) or directly inspecting volatile memory within the Local Security Authority Subsystem Service (`lsass.exe`).
* **Tooling:** Built-in `reg.exe`, `Invoke-Mimikatz.ps1` (`lsadump::sam`, `sekurlsa::logonpasswords`).

### Phase 4: Lateral Movement

* **Objective:** Traverse from the initial workstation to servers and domain infrastructure.
* **Mechanism:** Pass-the-Hash (PtH) leverages NTLM hashes directly within NTLM/Kerberos authentication exchanges without cracking plaintext passwords. Remote management is achieved over Windows Remote Management (WinRM) or WMI.
* **Tooling:** Mimikatz (`sekurlsa::pth`), PowerShell Remoting (`New-PSSession`, `Enter-PSSession`).

### Phase 5: Service Account Abuse (Kerberoasting)

* **Objective:** Extract domain credentials offline without triggering account lockout thresholds.
* **Mechanism:** Authenticated domain users request a Ticket Granting Service (TGS) ticket for any Service Principal Name (SPN). The returned ticket is encrypted with the service account's NTLM hash, allowing offline dictionary recovery.
* **Tooling:** `PowerView.ps1` (`Get-NetUser -SPN`), `klist`, `tgsrepcrack.py`, `hashcat`.

### Phase 6: Forest-Wide Persistence

* **Objective:** Maintain persistent administrative control that survives password rotations of standard administrative accounts.
* **Mechanism:** Compromising the `krbtgt` account hash via Directory Replication Service (DRS) remote protocol (DCSync). The stolen key allows the adversary to mathematically forge valid Ticket Granting Tickets (TGTs) containing arbitrary group memberships (e.g., Domain Admins, Enterprise Admins).
* **Tooling:** Mimikatz (`lsadump::dcsync`, `kerberos::golden`).

---

## 🛡️ 3. Defensive Telemetry & Detection Matrix

A structured detection strategy maps distinct Windows Security Event IDs and telemetry sources to each phase of the attack pathway:

| Attack Phase | Key Windows Event IDs | Telemetry Source | Defensive Countermeasure |
| --- | --- | --- | --- |
| **Pivoting / Ingress** | Event 4624 (Logon Type 10 - RemoteInteractive) | Network Firewalls, SSH Auth Logs | Segment DMZ networks; disable port forwarding on non-jumpbox hosts. |
| **Service Escalation** | Event 7040 (Service Start Type Change), Event 7045 (New Service Installed) | System Log, Sysmon Event 1 (Process Creation) | Harden service DACLs; restrict `SERVICE_CHANGE_CONFIG` to `SYSTEM` and Admins. |
| **Credential Dumping** | Sysmon Event 10 (ProcessAccess to `lsass.exe`), Event 4688 (`reg save`) | Process Memory Handles, Security Log | Enable LSA Protection (`RunAsPPL`), Credential Guard, and audit `reg.exe` exports. |
| **Lateral Movement** | Event 4624 (Logon Type 3 - Network), Event 4104 (Script Block Logging) | WinRM Operational Logs, PowerShell Logs | Restrict WinRM/SMB cross-workstation traffic; implement tiered administrative models. |
| **Kerberoasting** | Event 4769 (Kerberos Service Ticket Requested with RC4 Encryption `0x17`) | Security Log on Domain Controllers | Enforce Group Managed Service Accounts (gMSA); disable RC4; enforce 25+ character passwords. |
| **Golden Ticket** | Event 4662 (Directory Service Access targeting replication GUIDs), Event 4769 | Security Log on Domain Controllers | Regularly rotate the `krbtgt` password twice; restrict DCSync rights; monitor ticket lifetimes. |

