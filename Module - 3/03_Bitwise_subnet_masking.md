The operating system uses **bitwise subnet masking** against its **routing table** to determine whether a destination IP is local or remote.

---

**1. The Mathematical Check (Subnet Masking)**

Whenever Kali sends a packet, the kernel performs a logical `AND` operation between the **destination IP** and its own **subnet mask**:

* **Pinging Metasploitable (`192.168.50.3`):**
* Target IP: `192.168.50.3` $\rightarrow$ `11000000.10101000.00110010.00000011`
* Subnet Mask (`/24`): `255.255.255.0` $\rightarrow$ `11000000.10101000.00110010.00000000`
* Result: `192.168.50.0`
* **Match:** This matches your `eth1` network entry (`192.168.50.0/24`). The kernel flags the destination as **local (LAN)**.


* **Pinging Google (`142.250.190.46`):**
* Target IP: `142.250.190.46`
* Evaluated against `/24` subnet masks: Result is `142.250.190.0`.
* **No Match:** It does not match the local subnet of `eth0` or `eth1`. The kernel flags it as **remote (WAN)**.



---

**2. Longest Prefix Match in the Routing Table**

When you run `ip route show`, the kernel evaluates destinations from most specific to least specific:

| Routing Entry | Scope | Action Taken |
| --- | --- | --- |
| `192.168.50.0/24 dev eth1` | Specific local subnet | Matched for `192.168.50.3`. Packet stays on the local virtual switch via `eth1`. |
| `10.0.2.0/24 dev eth0` | Specific NAT subnet | Matched only for local VirtualBox NAT devices. |
| `default via 10.0.2.2 dev eth0` (`0.0.0.0/0`) | Catch-all for everything else | Matched for Google / internet IPs when no specific route exists. |

---

**3. Layer 2 Frame Construction (ARP vs. Gateway)**

Once the decision is made:

* **Local Target (LAN):** Kali sends an **ARP request** asking for the MAC address of `192.168.50.3` directly, encapsulating the IP packet in an Ethernet frame addressed directly to Metasploitable's virtual NIC.
* **Remote Target (Internet):** Kali keeps Google's IP as the destination IP, but encapsulates the frame with the MAC address of the **Default Gateway** (`10.0.2.2` via `eth0`), letting VirtualBox route it out to your physical router and the internet.