Your internet is failing because of two specific routing issues:

1. **Conflicting Default Gateway:** You defined `gateway 192.168.50.1` on `eth0` (Host-Only). Since `192.168.50.1` is just a local host-only virtual adapter on your computer, it has no route to the public internet. Kali is trying to send all external web traffic through that dead-end gateway instead of your Bridged adapter.


2. **The Bridged Interface is Not Configured:** Your `/etc/network/interfaces` only mentions `eth0`; the second adapter (`eth1`) is not configured to request an IP and gateway from your real home router via DHCP.



---

**Step 1: Fix `/etc/network/interfaces**`

1. In your open `nano` editor, remove the `gateway` line under `eth0` (Host-Only interfaces should never have a default gateway when paired with an internet adapter).
2. Add the configuration for `eth1` (the Bridged adapter) using `dhcp`.

Update the file so it looks like this:

```text
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# Host-Only Adapter (Lab Network)
auto eth0
iface eth0 inet static
    address 192.168.50.2
    netmask 255.255.255.0

# Bridged Adapter (Internet Access)
auto eth1
iface eth1 inet dhcp

```

Save the file (`Ctrl + O`, `Enter`, then `Ctrl + X`).

---

**Step 2: Apply the Changes and Request an IP**

Run the following commands in the terminal:

```bash
sudo systemctl restart networking
sudo dhclient -v eth1

```

---

**Step 3: Verify the Default Route and Connectivity**

1. Check your routing table:
```bash
ip route show

```


*Make sure the `default via ...` route points to `dev eth1` (your home router's IP), NOT `eth0`.*
2. Test DNS and external connectivity:
```bash
ping -c 3 8.8.8.8
ping -c 3 google.com

```











The problem is that your **interfaces are inverted**.

Looking closely at your `ifconfig` output:

* **`eth1`** has MAC address **`08:00:27:c1:5b:a9`**, which matches **Host-Only Adapter #3** from your VirtualBox settings (where DHCP is disabled).
* **`eth0`** has MAC address **`08:00:27:cb:d3:18`**, which is your **Bridged / Internet adapter**.

Because `eth0` was configured with the static lab IP and `eth1` with DHCP, `eth1` is broadcasting `DHCPDISCOVER` to a Host-Only adapter with no DHCP server, while your actual internet adapter (`eth0`) is locked to the offline `192.168.50.2` address.

---

**Step 1: Swap the Configuration in `/etc/network/interfaces**`

Open the file:

```bash
sudo nano /etc/network/interfaces

```

Update it so **`eth0` gets DHCP** (Internet) and **`eth1` gets the Static IP** (Host-Only Lab):

```text
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# Bridged Adapter (Internet Access)
auto eth0
iface eth0 inet dhcp

# Host-Only Adapter (Lab Network)
auto eth1
iface eth1 inet static
    address 192.168.50.2
    netmask 255.255.255.0

```

Save and exit (`Ctrl + O`, `Enter`, `Ctrl + X`).

---

**Step 2: Restart Networking and Request IP for `eth0**`

Flush the previous static IP from `eth0` and request a DHCP lease from your router:

```bash
sudo ip addr flush dev eth0
sudo systemctl restart networking
sudo dhclient -v eth0

```

---

**Step 3: Verify Connectivity**

1. Check your IP addresses and routes:
```bash
ip a
ip route show

```


*(You should see an IP from your home router on `eth0` and a default route via `eth0`)*
2. Test external connectivity:
```bash
ping -c 3 8.8.8.8
ping -c 3 google.com

```









To fix this and get internet working on your Kali VM without breaking your lab network, switch the internet adapter from **Bridged** to **NAT** in VirtualBox:

---

**Step 1: Change Adapter Mode to NAT in VirtualBox**

1. In the top VirtualBox VM window menu, click **Devices** $\rightarrow$ **Network** $\rightarrow$ **Network Settings...**
2. Check the adapters to find the one matching MAC address `08:00:27:cb:d3:18` (this is mapped to `eth0`).
3. Change **Attached to:** from `Bridged Adapter` to **`NAT`**.
4. Expand **Advanced** and verify that **Cable Connected** is checked.
5. Click **OK**.

---

**Step 2: Pull an IP Address on `eth0**`

Run the DHCP client again in your Kali terminal:

```bash
sudo dhclient -v eth0

```

VirtualBox's internal NAT service will immediately assign an IP (such as `10.0.2.15`) and install a default gateway route.

---

**Step 3: Verify Routes and DNS**

1. **Check the routing table:**
```bash
ip route show

```


You should see a line starting with `default via 10.0.2.2 dev eth0`, while `eth1` maintains `192.168.50.0/24 dev eth1`.


2. **Test connectivity:**
```bash
ping -c 3 8.8.8.8
ping -c 3 google.com

```