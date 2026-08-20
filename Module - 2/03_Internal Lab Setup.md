# 🏢 AD Red Team Lab — Internal Network & Domain Setup

## 📐 1. Internal Network Architecture

The internal Active Directory network operates within an isolated subnet (`10.10.10.0/24`) connected via **VirtualBox Host-Only Ethernet Adapter #4**. It hosts the identity provider (DC), an enterprise application server, and an end-user client workstation.

```text
+---------------------------------------------------------------------------------------------------+
|                     INTERNAL CORPORATE NETWORK (Host-Only Adapter #4: 10.10.10.0/24)              |
+---------------------------------------------------------------------------------------------------+
       |                                           |                                         |
       v                                           v                                         v
+--------------------------+             +--------------------------+             +--------------------------+
|    Domain Controller     |             |    Application Server    |             |   Employee Workstation   |
|   [Windows Server 2016]  |             |   [Windows Server 2012]  |             |       [Windows 7]        |
|                          |             |                          |             |                          |
|  Adapter 4 (eth0)        | <---------> |  Adapter 4 (eth0)        | <---------> |  Adapter 4 (eth0)        |
|  IP:  10.10.10.2/24      |  (Kerberos/ |  IP:  10.10.10.3/24      |  (HTTP/RPC/ |  IP:  10.10.10.4/24      |
|  DNS: 10.10.10.2         |    LDAP)    |  DNS: 10.10.10.2         |    Auth)    |  DNS: 10.10.10.2         |
|  GW:  10.10.10.1         |             |  GW:  10.10.10.1         |             |  GW:  10.10.10.1         |
+--------------------------+             +--------------------------+             +--------------------------+
```

---

## 📊 2. Machine & Credential Inventory

| Machine Role | Operating System | VirtualBox Adapter | Static IP Address | Subnet Mask | Default Gateway | Primary DNS | Username | Default Password |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Domain Controller** | Windows Server 2016 | Host-Only Adapter #4 | `10.10.10.2` | `255.255.255.0` (`/24`) | `10.10.10.1` | `10.10.10.2` (or `127.0.0.1`) | `dc_user` | `******` |
| **Application Server** | Windows Server 2012 | Host-Only Adapter #4 | `10.10.10.3` | `255.255.255.0` (`/24`) | `10.10.10.1` | `10.10.10.2` | `app_user` | `******` |
| **Employee Workstation** | Windows 7 | Host-Only Adapter #4 | `10.10.10.4` | `255.255.255.0` (`/24`) | `10.10.10.1` | `10.10.10.2` | `karlos` | `windows7` |

---

## ⚙️ 3. Static IP Configuration Methods

### Option A: GUI Method (Windows 7 / Desktop Experience)

1. Open **Control Panel** -> **Network and Internet** -> **Network and Sharing Center**.
2. Click on **Change adapter settings** (or click the active **Local Area Connection / Ethernet** link).
3. Right-click the interface and select **Properties**.
4. Select **Internet Protocol Version 4 (TCP/IPv4)** and click **Properties**.
5. Select **Use the following IP address** and enter the static parameters (IP, Subnet Mask, Gateway).
6. Select **Use the following DNS server addresses** and point DNS to the Domain Controller (`10.10.10.2`).
7. Click **OK** to save and apply.

---

### Option B: CLI / PowerShell Method (Server Core & Automation)

#### 1. Using PowerShell (Recommended for Server 2012/2016)

```powershell
# Identify interface name/alias
Get-NetAdapter

# Configure Static IP, Subnet Mask, and Gateway (Replace "Ethernet" with your alias)
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.10.10.X -PrefixLength 24 -DefaultGateway 10.10.10.1

# Configure DNS Server (Point to Domain Controller 10.10.10.2)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("10.10.10.2")
```

#### 2. Using Classic Command Prompt (`netsh` for Windows 7 / CMD)

```cmd
:: Configure Static IP, Subnet Mask, and Gateway
netsh interface ipv4 set address name="Local Area Connection" static 10.10.10.X 255.255.255.0 10.10.10.1

:: Set Primary DNS Server
netsh interface ipv4 set dns name="Local Area Connection" static 10.10.10.2 primary
```

---

## 🛡️ 4. Firewall & Connectivity Configuration

Windows Server blocks incoming ICMP (ping) echo requests by default. Enable ICMP across all machines so connectivity can be verified:

```powershell
# Allow incoming ping via PowerShell / CMD
netsh advfirewall firewall add rule name="Allow ICMPv4-In" protocol=icmpv4:8,any dir=in action=allow

# OR completely disable firewall in the isolated lab environment
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```

---

## 🔍 5. Verification & Connectivity Tests

### From Workstation / App Server:
```cmd
:: 1. Verify assigned IP details
ipconfig /all

:: 2. Ping Domain Controller
ping -n 3 10.10.10.2

:: 3. Test DNS Resolution once AD is configured
nslookup dc.lab.local 10.10.10.2
```