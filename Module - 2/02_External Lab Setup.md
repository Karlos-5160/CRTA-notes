# 🛡️ AD Red Team Lab — Network & VM Architecture Setup

## 📐 1. Network Topology & Subnet Design

The lab simulates an enterprise DMZ pivot scenario where an external attacker breaches a dual-homed perimeter web server to gain access to the isolated internal Active Directory network.
```text
       [ ATTACKER NETWORK ]                  [ DMZ / EXTERNAL ]                      [ INTERNAL AD NETWORK ]
  (Host-Only Adapter #3: 192.168.50.0/24)                                  (Host-Only Adapter #4: 10.10.10.0/24)
  
  +--------------------------+             +--------------------------+             +--------------------------+
  |       Kali Linux         |             |   Web-Server (Dual-NIC)  |             |     Employee Machine     |
  |      [Attacker VM]       |             |     [Metasploitable]     |             |      [Internal Target]   |
  |                          |             |                          |             |                          |
  |  Adapter 3 (eth0 / eth1) | <---------> |  Adapter 3 (eth0)        |             |  Adapter 4 (eth0)        |
  |  IP: 192.168.50.2/24     |  (Host-Only |  IP: 192.168.50.3/24     |             |  IP: 10.10.10.x/24       |
  |  GW: 192.168.50.1        |     #3)     |  GW: 192.168.50.1        |             |  GW: 10.10.10.1          |
  +--------------------------+             +------------+-------------+             +------------+-------------+
                                                        |                                        |
                                                        |  Adapter 4 (eth1)                      |
                                                        |  IP: 10.10.10.5/24 (Host-Only #4)      |
                                                        +----------------------------------------+
```
---

## 📊 2. Subnet & IP Allocation Matrix

| Machine Role | VirtualBox Adapter Name | Interface Name | Assigned IP Address | Subnet Mask | Gateway | Network Domain |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Attacker (Kali Linux)** | Host-Only Adapter #3 | `eth0` / `eth1` | `192.168.50.2` | `255.255.255.0` (`/24`) | `192.168.50.1` | External Attacker Network |
| **Web Server (Metasploitable)** | Host-Only Adapter #3 | `eth0` | `192.168.50.3` | `255.255.255.0` (`/24`) | `192.168.50.1` | External / DMZ Network |
| **Web Server (Metasploitable)** | Host-Only Adapter #4 | `eth1` | `10.10.10.5` | `255.255.255.0` (`/24`) | *None (Direct)* | Internal AD Subnet |
| **Employee Target Machine** | Host-Only Adapter #4 | `eth0` | `10.10.10.x` | `255.255.255.0` (`/24`) | `10.10.10.1` | Internal Corporate LAN |

---

## ⚙️ 3. VirtualBox Host-Only Configuration

Ensure the VirtualBox Host-Only adapters are configured via **VirtualBox Manager -> Tools -> Network**:

1. **Adapter #3 (External Subnet):**
   * **IPv4 Prefix:** `192.168.50.0/24`
   * **IPv4 Address:** `192.168.50.1`
   * **DHCP Server:** `Disabled` (Manual static assignment)
2. **Adapter #4 (Internal Subnet):**
   * **IPv4 Prefix:** `10.10.10.0/24`
   * **IPv4 Address:** `10.10.10.1`
   * **DHCP Server:** `Disabled` (Manual static assignment)

---

## 🖥️ 4. Machine Configurations

### A. Attacker Machine (Kali Linux)

#### 1. Hardware Settings (VirtualBox)
* **Adapter Settings:** Attach **Network Adapter** to `VirtualBox Host-Only Ethernet Adapter #3`.
* **Promiscuous Mode:** `Allow VMs` (or `Allow All` for sniffing/spoofing exercises).

#### 2. Network Configuration
Edit `/etc/network/interfaces` or apply directly using `ip`:

```bash
# Append to /etc/network/interfaces
auto eth0
iface eth0 inet static
    address 192.168.50.2
    netmask 255.255.255.0
    gateway 192.168.50.1
    dns-nameservers 1.1.1.1 8.8.8.8