# I am not using proxychains then how am i able to ping the machine in the internal network [ping 10.10.10.2] ?
The ping is succeeding without Proxychains because of **VirtualBox routing leakage through your physical Windows host**:

---

**1. VirtualBox Creates a Host Interface on Windows**
When you created the Host-Only network `Host-Only Ethernet Adapter #4` for the internal `10.10.10.0/24` lab subnet on VirtualBox, Windows automatically created a virtual network adapter on your host machine (usually with IP `10.10.10.1`).

---

**2. The Egress Path via NAT (`eth0`)**
Look at your `ifconfig`:

* `eth0` is using **NAT** (`10.0.2.15`).
* Kali's default gateway is `10.0.2.2` (the VirtualBox NAT engine running inside your Windows host OS).



When you run `ping 10.10.10.2` on Kali without Proxychains:

1. Kali's kernel checks its routing table: `10.10.10.2` does not match `192.168.50.0/24`, so it sends the ICMP packet out through its default gateway (`eth0` $\rightarrow$ `10.0.2.2`).


2. The packet hits the **VirtualBox NAT engine on your Windows host**.
3. Your **Windows host OS** checks its own routing table, sees that `10.10.10.0/24` is directly attached to its own `Host-Only Network Adapter #4`, and forwards the ping packet straight across to the Domain Controller VM (`10.10.10.2`).


4. The Domain Controller replies (`ttl=127`, indicating a Windows host response), and VirtualBox NAT translates it back to Kali.

---

**3. How to Make the Lab Truly Isolated (Realistic Red Team Setup)**

To prevent your Windows host from routing traffic between Kali and the internal network:

1. **Option A: Remove Host Routing (Host-Only Network Settings)**
* In VirtualBox, go to **Tools $\rightarrow$ Network Manager $\rightarrow$ Host-only Networks**.
* Select `Host-Only Adapter #4` and uncheck the DHCP/Host interface, or switch the internal lab adapters to **Internal Network** instead of **Host-Only**.


2. **Option B: Use "Internal Network" Mode**
* Change Adapter 2 on Metasploitable, Adapter 1 on DC (`10.10.10.2`), and Adapter 1 on Windows 7 (`10.10.10.4`) to **Internal Network** (e.g., name it `intnet`).


* This completely isolates the internal segment so that **only** the dual-homed Metasploitable machine can bridge traffic between subnets, making pivoting mandatory.