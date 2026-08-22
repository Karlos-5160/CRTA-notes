# External Red team Operations
    • As an attacker [192.168.50.2] we will try to get initial foothold to the Web-Server
    [192.168.50.3]
    • We will use Web as well as Network related vulnerabilities to get access to the Web-
    Server.

## External exposed service exploitation
    • It is the exploitation of the services in the web or network which can be
    easily accessed.
    • Adversary makes an active connection with the externally exposed server to
    identify and leverage existing loopholes.
    • Adversary with adequate knowledge of target organization's infrastructure
    and technologies identify the weakest link and try to exploit it.
    • Externally exposed service can exist in Web or in the network.

## NetDiscover
    • NetDiscover is a very neat tool for finding hosts on either wireless or switched networks.
``` bash
    netdiscover -i <interface> -r <IP address CIDR format>
```

## Vulnerability Assessment
    • Vulnerability assessment is a systematic review of security weaknesses in an
    information system.
    • It evaluates if the system is susceptible to any known vulnerabilities, assigns severity
    levels to those vulnerabilities, and recommends remediation or mitigation, if and
    whenever needed.
    • It follows a 4-step process :
    Vuln Identification
    Analysis
    Risk Assessment
    Remediation

``` text
# A lot of tools exists in the market for the Vulnerability Assessment process :
    Nessus
    Acunetix
    Qualys Vulnerability Management
    Netsparker
    Metasploit
    Amazon Inspector (ONLY for applications deployed on AWS)
```
    • For understanding the vulnerability scanning process, one can also use Nikto (for web
    application) and nmap (for both Network as well as web app) vulnerability scanning.