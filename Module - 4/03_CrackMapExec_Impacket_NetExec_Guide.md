# CrackMapExec, Impacket, and NetExec
## 1. Quick overview

| Tool | What it is | Best way to think about it |
|---|---|---|
| **Impacket** | A Python library and collection of protocol-focused tools | A toolbox for SMB, Kerberos, LDAP, RPC, and other Windows/AD protocols |
| **CrackMapExec (CME)** | A framework for assessing multiple hosts and services | A convenient way to run checks and enumeration across hosts |
| **NetExec (NXC)** | A modern continuation/evolution of the CME-style workflow | The newer tool commonly used for similar network and AD assessment tasks |

A simple mental model:

```text
Nmap       -> What hosts and services are available?
NetExec/CME-> What can I learn or do across those services with authorized access?
Impacket   -> Which lower-level protocol tool do I need for a specific task?
```

---

# 2. Impacket

## What is Impacket?

Impacket is a collection of Python classes and command-line tools for working with network protocols commonly found in Windows and Active Directory environments.

Common protocols include:

- SMB
- MSRPC / DCE-RPC
- Kerberos
- LDAP
- NTLM

Impacket is especially useful when you need a **specific protocol-focused tool**.

## Useful Impacket commands

> Command names can differ slightly depending on your installation and version. On Kali/Debian packages, many commands are prefixed with `impacket-`.

### A. View help

```bash
impacket-smbclient -h
```

### B. Connect to an SMB service with authorized credentials

```bash
impacket-smbclient DOMAIN/username:password@192.168.1.10
```

Once connected, you can inspect resources available to that authorized account.

### C. Query basic information from SMB

```bash
impacket-lookupsid DOMAIN/username:password@192.168.1.10
```

This can be used in an authorized environment to query SID-related account information.

### D. Query Active Directory service principal names

```bash
impacket-GetUserSPNs DOMAIN/username:password -dc-ip 192.168.1.10
```

This is useful for understanding service accounts and SPNs in an authorized AD assessment.

### E. Check for accounts configured without Kerberos pre-authentication

```bash
impacket-GetNPUsers DOMAIN/username:password -dc-ip 192.168.1.10
```

Use this only against environments you are authorized to assess.

### F. Remote administration examples

Impacket also contains tools such as:

```bash
impacket-psexec -h
impacket-wmiexec -h
impacket-smbexec -h
```

These tools are used for remote administration or security testing when you have valid authorization and appropriate credentials.

## When should I use Impacket?

Use Impacket when:

- You want direct interaction with SMB, Kerberos, LDAP, or RPC.
- You need a specific AD/Windows protocol utility.
- You are following a CTF or lab workflow that calls for a particular tool such as `GetUserSPNs`, `GetNPUsers`, or `smbclient`.

---

# 3. CrackMapExec (CME)

## What is CrackMapExec?

CrackMapExec is a framework designed to make it easier to assess multiple hosts using protocols such as:

- SMB
- WinRM
- LDAP
- MSSQL
- SSH

It is often used to quickly gather information and test authorized credentials across a group of machines.

The basic pattern is:

```text
crackmapexec <protocol> <target> <options>
```

## Useful CME commands

### A. Display help

```bash
crackmapexec -h
```

Protocol-specific help:

```bash
crackmapexec smb -h
```

### B. Check an SMB service

```bash
crackmapexec smb 192.168.1.10
```

This can display basic information exposed by the SMB service.

### C. Authenticate with authorized credentials

```bash
crackmapexec smb 192.168.1.10 -u username -p password
```

### D. Check multiple lab hosts

```bash
crackmapexec smb 192.168.1.0/24 -u username -p password
```

This is useful in a lab where you own or are explicitly authorized to test the hosts.

### E. Check accessible shares

```bash
crackmapexec smb 192.168.1.10 -u username -p password --shares
```

### F. Use another supported protocol

For example, WinRM:

```bash
crackmapexec winrm 192.168.1.10 -u username -p password
```

LDAP:

```bash
crackmapexec ldap 192.168.1.10 -u username -p password
```

## When should I use CME?

Use CME when:

- You have multiple machines to assess.
- You want fast SMB/WinRM/LDAP enumeration.
- You want to check whether authorized credentials work across a set of lab machines.
- You want a high-level assessment workflow rather than separate protocol tools.

---

# 4. NetExec (NXC)

## What is NetExec?

NetExec is a modern tool with a workflow very similar to CrackMapExec. It is commonly used as the newer alternative in current labs and security assessments.

The basic syntax is:

```text
nxc <protocol> <target> <options>
```

If you already know CME, NetExec will feel familiar.

## Useful NetExec commands

### A. Display general help

```bash
nxc -h
```

Protocol-specific help:

```bash
nxc smb -h
```

### B. Inspect an SMB service

```bash
nxc smb 192.168.1.10
```

### C. Authenticate with authorized credentials

```bash
nxc smb 192.168.1.10 -u username -p password
```

### D. Check multiple authorized lab systems

```bash
nxc smb 192.168.1.0/24 -u username -p password
```

### E. Enumerate accessible shares

```bash
nxc smb 192.168.1.10 -u username -p password --shares
```

### F. WinRM example

```bash
nxc winrm 192.168.1.10 -u username -p password
```

### G. LDAP example

```bash
nxc ldap 192.168.1.10 -u username -p password
```

## When should I use NetExec?

Use NetExec when:

- You are learning current Active Directory assessment workflows.
- A modern CME-style tool is available in your environment.
- You want to assess multiple authorized systems through SMB, LDAP, WinRM, MSSQL, or other supported protocols.

---

# 5. CME vs NetExec vs Impacket

| Feature | Impacket | CrackMapExec | NetExec |
|---|---|---|---|
| Main style | Protocol-focused tools | Multi-host assessment framework | Modern multi-host assessment framework |
| Typical use | Specific SMB/Kerberos/RPC/AD tasks | Fast checks across multiple hosts | Similar workflow with modern development |
| Example command | `impacket-smbclient` | `crackmapexec smb` | `nxc smb` |
| Best for | Fine-grained tasks | Automation and broad assessment | Current CME-style workflows |

## Important difference

Suppose you are working in an authorized Active Directory lab.

You discover several Windows hosts.

### Step 1: Discover services

```bash
nmap -sV 192.168.1.0/24
```

### Step 2: Quickly inspect SMB hosts

With CME:

```bash
crackmapexec smb 192.168.1.0/24
```

Or with NetExec:

```bash
nxc smb 192.168.1.0/24
```

### Step 3: Use a specific protocol-focused utility

For example:

```bash
impacket-smbclient DOMAIN/username:password@192.168.1.10
```

So the workflow can be understood as:

```text
Nmap
  |
  v
Find hosts and services
  |
  v
CME / NetExec
  |
  v
Quickly assess multiple authorized targets
  |
  v
Impacket
  |
  v
Perform a specific SMB / Kerberos / RPC / AD task
```

---

# 6. Recommended learning order

For Active Directory labs and CTFs, a good order is:

1. Learn SMB, LDAP, Kerberos, NTLM, and WinRM basics.
2. Learn common Impacket tools:
   - `smbclient`
   - `lookupsid`
   - `GetUserSPNs`
   - `GetNPUsers`
   - `psexec`
   - `wmiexec`
3. Learn NetExec for efficient enumeration across authorized lab machines.
4. Learn CME because many older writeups and tutorials still use its syntax.
5. Practice by reading each command's help:

```bash
nxc smb -h
impacket-smbclient -h
impacket-GetUserSPNs -h
crackmapexec smb -h
```

---

# 7. Final summary

- **Impacket** = a collection of specialized tools for Windows and Active Directory protocols.
- **CrackMapExec** = a framework for quickly assessing multiple systems.
- **NetExec** = a modern CME-style alternative commonly used in current security labs.
- **Nmap + NetExec/CME + Impacket** is a useful combination for authorized Windows/Active Directory testing.
