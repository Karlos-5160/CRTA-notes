# 🌐 Linux Dual-NIC Architecture & OS Routing Breakdown

## 🔌 1. Hardware Interface Mapping & Configuration

### Virtual NIC Mapping
* VirtualBox assigns a virtual PCIe slot and a unique MAC address to every enabled network adapter.
* The Linux kernel registers each physical/virtual NIC as an interface (`eth0`, `eth1`, etc.) bound to that specific MAC address.

### Dual-Adapter Configuration (`/etc/network/interfaces`)
* **`eth0` (Internet Access via NAT):** Uses DHCP to obtain an IP address, DNS servers, and the single system **Default Gateway** (`0.0.0.0/0`).
* **`eth1` (Internal Lab Subnet via Host-Only):** Uses a static IP for the isolated lab segment without defining a gateway, preventing default route collisions.

```text
source /etc/network/interfaces.d/*

# Loopback Interface
auto lo
iface lo inet loopback

# Internet Adapter (NAT)
auto eth0
iface eth0 inet dhcp

# Lab Host-Only Adapter
auto eth1
iface eth1 inet static
    address 192.168.50.2
    netmask 255.255.255.0

```

---

## 🧮 2. Packet Routing Decision (LAN vs. WAN Determination)

When an outbound packet is generated, the kernel determines its path via **Bitwise Subnet Masking** and **Longest Prefix Matching**:

```text
+---------------------------------------------------------------------------------------------------+
| 1. HARDWARE TO INTERFACE MAPPING                                                                  |
|    VirtualBox Virtual NIC (MAC Address) <---> Linux Interface (eth0 / eth1)                       |
+---------------------------------------------------------------------------------------------------+
                                                  |
                                                  v
+---------------------------------------------------------------------------------------------------+
| 2. BITWISE SUBNET MASKING CHECK                                                                   |
|    Target IP  AND  Interface Subnet Mask  ==  Local Subnet ID?                                    |
+---------------------------------------------------------------------------------------------------+
                  |                                                  |
       [ MATCH (Local Subnet) ]                           [ NO MATCH (Remote / WAN) ]
                  |                                                  |
                  v                                                  v
+------------------------------------+             +------------------------------------+
| 3A. LOCAL LAN DELIVERY             |             | 3B. DEFAULT GATEWAY ROUTING        |
| - Matches eth1 (192.168.50.0/24)   |             | - Falls back to default 0.0.0.0/0  |
| - Kernel sends ARP for Target IP   |             | - Kernel sends ARP for Gateway MAC |
| - Frame addressed directly to VM   |             | - Frame sent to NAT Router (eth0)  |
+------------------------------------+             +------------------------------------+

```

---

## 📊 3. Routing Mechanism Comparison

| Evaluation Metric | Local LAN Path (`eth1`) | Remote WAN / Internet Path (`eth0`) |
| --- | --- | --- |
| **Example Target** | Metasploitable (`192.168.50.3`) | Google (`142.250.190.46`) |
| **Bitwise AND Result** | `192.168.50.0` (Matches `eth1` network ID) | `142.250.190.0` (No local match) |
| **Routing Table Match** | `192.168.50.0/24 dev eth1` | `default via 10.0.2.2 dev eth0` (`0.0.0.0/0`) |
| **Layer 2 Encapsulation** | Destination MAC = Target VM MAC | Destination MAC = Default Gateway (Router) MAC |
| **ARP Target** | Destination Host IP | Gateway Router IP |
