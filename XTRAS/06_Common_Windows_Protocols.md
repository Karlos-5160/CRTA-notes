## Big picture

| Technology | Full form                             | Main purpose                                      | Typical port(s)                        |
| ---------- | ------------------------------------- | ------------------------------------------------- | -------------------------------------- |
| **RDP**    | Remote Desktop Protocol               | Remote graphical desktop                          | **TCP/UDP 3389**                       |
| **WinRM**  | Windows Remote Management             | Remote command/management                         | **5985 HTTP, 5986 HTTPS**              |
| **WMI**    | Windows Management Instrumentation    | Query/manage Windows systems                      | **135 + dynamic RPC ports**            |
| **WS**     | Web Services / Web Service            | Communication through web protocols               | Usually **80/443**, depends on service |
| **LDAP**   | Lightweight Directory Access Protocol | Query/manage directory services such as AD        | **389**, LDAPS **636**                 |
| **SMB**    | Server Message Block                  | File/printer sharing and Windows network services | **445**, older **139**                 |

---

# 1. RDP — Remote Desktop Protocol

**RDP lets you remotely control a Windows computer with a graphical desktop.**

For example:

```text
Your PC
   │
   │ RDP :3389
   ▼
Windows Server
   │
   └── You see its desktop
```

You can:

* See the remote Windows desktop
* Use the mouse and keyboard
* Run applications
* Access files
* Perform administrative tasks if your account has permission

### Example

From Windows:

```text
mstsc
```

Then enter:

```text
192.168.1.10
```

From Linux, tools such as `xfreerdp` can be used for legitimate remote administration.

### Remember

**RDP = “Give me the remote Windows desktop.”**

---

# 2. WinRM — Windows Remote Management

WinRM is designed for **remote management of Windows machines**, particularly through command-line/management interfaces rather than giving you a full desktop.

Default ports:

```text
5985 → HTTP
5986 → HTTPS
```

For example, PowerShell Remoting uses WinRM.

Conceptually:

```text
Administrator
     │
     │ WinRM
     ▼
Windows Server
     │
     ├── PowerShell
     ├── Commands
     └── Management operations
```

Example PowerShell:

```powershell
Enter-PSSession -ComputerName SERVER01
```

You can then interact with the remote PowerShell session.

### Remember

**WinRM = “Manage the Windows machine remotely.”**

---

# 3. WMI — Windows Management Instrumentation

WMI is a **Windows management/information framework**.

It allows software and administrators to retrieve information about a Windows system and perform management operations.

For example, you can query things like:

```text
What processes are running?
What services exist?
What operating system version?
What users exist?
What hardware is installed?
What network configuration exists?
```

A simplified picture:

```text
Management Tool
       │
       │ WMI
       ▼
Windows Management Infrastructure
       │
       ├── Processes
       ├── Services
       ├── Hardware
       ├── OS information
       └── Configuration
```

WMI commonly uses **DCOM/RPC**, so you'll often see:

```text
TCP 135
+
dynamic RPC ports
```

### WMI vs WinRM

This is an important distinction:

**WMI** is primarily a **management/instrumentation framework**.

**WinRM** is a **remote management transport/protocol**.

They can overlap because Windows management tools can use different mechanisms to perform remote operations.

---

# 4. WS Management (WS Man)

**WS-Management is a web-services-based protocol used to remotely manage computers and devices.**

In Windows, it is mainly associated with **WinRM**.

Think of it as:

> **WS-Man = the communication protocol; WinRM = Microsoft's Windows implementation of it.**

It can be used for:

* Remote PowerShell/command management
* Querying system information
* Managing Windows services and resources

Common ports:

* **5985 → HTTP**
* **5986 → HTTPS**

Simple flow:

```text
Your PC
   │
   │ WS-Man
   ▼
 WinRM
   │
   ▼
Remote Windows Machine
```

**Exam one-liner:**
**WS-Man is a SOAP/web-services-based protocol for remote management of computers and devices.**

---

# 5. LDAP — Lightweight Directory Access Protocol

LDAP is very important when you're studying **Active Directory**.

LDAP is used to communicate with a **directory service**.

In a Windows domain, that directory service is **Active Directory Domain Services (AD DS)**.

Imagine an organization's directory:

```text
example.local
│
├── Users
│   ├── Alice
│   ├── Bob
│   └── Administrator
│
├── Computers
│   ├── PC01
│   └── PC02
│
├── Groups
│   ├── Domain Admins
│   └── Employees
│
└── Organizational Units
```

LDAP allows applications to query this directory.

For example:

```text
"Find all users in this directory"

"Find the Domain Admins group"

"Find this computer"

"Give me information about this user"
```

Typical ports:

```text
389  → LDAP
636  → LDAPS
```

### Remember

**LDAP = “Talk to the directory.”**

---

# 6. SMB — Server Message Block

SMB is one of the most important Windows networking protocols.

Its most recognizable purpose is **network file sharing**.

For example:

```text
PC A
 │
 │ SMB
 ▼
\\SERVER01\SharedFiles
```

You might see Windows paths such as:

```text
\\192.168.1.10\Shared
```

or:

```text
\\SERVER01\Users
```

SMB can also support things such as:

* File sharing
* Printer sharing
* Named pipes
* Various Windows network operations

Modern SMB commonly uses:

```text
TCP 445
```

Older Windows networking could involve:

```text
TCP 139
```

### Remember

**SMB = “Access resources shared over the Windows network.”**

---

# 7. NetBIOS

**NetBIOS (Network Basic Input/Output System)** is a legacy software interface and application programming interface (API) that allows computers on a local network to **identify each other, share files or printers and communicate with each other using computer names**.

Think of it as:

> **NetBIOS = an old way for Windows computers to discover and communicate with each other by name.**

### Main NetBIOS services

NetBIOS traditionally uses three services:

| Service                         | Purpose                                |        Port |
| ------------------------------- | -------------------------------------- | ----------: |
| **NetBIOS Name Service (NBNS)** | Resolves computer names → IP addresses | **UDP 137** |
| **NetBIOS Datagram Service**    | Connectionless communication           | **UDP 138** |
| **NetBIOS Session Service**     | Establishes sessions between computers | **TCP 139** |

### NetBIOS and SMB

This is where it can get confusing:

```text
Older Windows networking:

SMB
 ↓
NetBIOS
 ↓
TCP 139
```

Modern SMB generally works directly over:

```text
SMB
 ↓
TCP 445
```

So when you see:

```text
139/tcp open netbios-ssn
445/tcp open microsoft-ds
```

you should generally think:

**139 → SMB over NetBIOS**

**445 → SMB directly over TCP**

### NetBIOS vs DNS

They're both related to names, but they're different:

```text
DNS
  ↓
www.example.com → IP address

NetBIOS
  ↓
COMPUTER01 → IP address
```

**In short:**

> **NetBIOS is an older Windows networking system for computer-name resolution and communication, commonly associated with ports 137–139.**

---

# 8.MS-RPC (Microsoft Remote Procedure Call)

**MS-RPC is a Microsoft protocol that allows a program on one computer to call and execute functions on another computer over a network.**

Think of it as:

> **“Run this function on that remote Windows machine.”**

It is heavily used by **Windows and Active Directory** for services such as remote management, authentication-related operations, and system administration.

### Important port

```text
TCP 135 → RPC Endpoint Mapper
```

After contacting **135**, the client may be directed to a **dynamic RPC port** where the actual RPC service is running.

### Simple flow

```text
Client
  │
  │ RPC request
  ▼
TCP 135 (Endpoint Mapper)
  │
  ▼
Dynamic RPC port
  │
  ▼
Windows service
```

**Exam one-liner:**
**MS-RPC enables Windows applications and services to perform remote procedure calls between computers over a network.**

---

# The easiest way to differentiate all six

Imagine you have a Windows server called:

```text
SERVER01
```

You want to do different things with it.

### 🖥️ "I want to see and control its desktop."

Use:

**RDP**

```text
Your PC ───── RDP ─────> SERVER01
                         🖥️
```

---

### ⌨️ "I want a remote PowerShell/management session."

Use:

**WinRM**

```text
Your PC ───── WinRM ────> SERVER01
                         PowerShell
```

---

### 🔎 "I want information about the Windows system."

Use:

**WMI**

```text
Tool ───── WMI ─────> SERVER01
                      │
                      ├── Processes
                      ├── Services
                      └── System information
```

---

### 👥 "I want to query Active Directory."

Use:

**LDAP**

```text
Tool ───── LDAP ─────> Domain Controller
                       │
                       ├── Users
                       ├── Groups
                       ├── Computers
                       └── OUs
```

---

### 📁 "I want to access a Windows network share."

Use:

**SMB**

```text
Your PC ───── SMB ────> SERVER01
                       │
                       └── \\SERVER01\Share
```

---

### 🌐 "I want applications to communicate through web protocols."

Use:

**Web Services**

```text
Application A ── HTTP/HTTPS ──> Web Service
```

---

# Ports cheat sheet

For cybersecurity/CTF work, this is worth memorizing:

```text
21    FTP
22    SSH

25    SMTP

53    DNS

80    HTTP
88    Kerberos

110   POP3
135   MSRPC

139   NetBIOS/SMB
389   LDAP
443   HTTPS
445   SMB

464   Kerberos password change
636   LDAPS

1433  MSSQL

3268  Global Catalog LDAP
3269  Global Catalog LDAPS

3389  RDP

5985  WinRM HTTP
5986  WinRM HTTPS
```

### A useful mental map

```text
                    WINDOWS / AD
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
      RDP              WinRM              SMB
       │                 │                 │
   GUI desktop      Remote management   File sharing
       │                 │                 │
     :3389          :5985/:5986          :445
       │
       │
       └───────────────┐
                       │
                      WMI
                       │
               Windows management
                 :135 + RPC
                       
                  ACTIVE DIRECTORY
                       │
                      LDAP
                       │
                Directory queries
                    :389/:636
```

**For your cybersecurity studies, the three distinctions I'd memorize first are:**

> **RDP → GUI**
> **WinRM → Remote Windows management/PowerShell**
> **SMB → Files/shares and Windows network resources**
> **WMI → Windows system management/information**
> **LDAP → Active Directory directory queries**
> **WS-Man → Web-services management protocol used by WinRM**
