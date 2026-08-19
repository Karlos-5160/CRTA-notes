`/etc/network/interfaces` is a **Linux configuration file used to define how network interfaces should be configured**.

On systems that use the traditional **ifupdown** networking system (common in older Debian/Kali setups), it tells Linux things like:

* Which network interfaces exist
* Whether they should start automatically
* Whether they use **DHCP** or a **static IP**
* IP address, subnet mask, gateway
* DNS-related configuration in some setups

### Example

You might see:

```text
auto eth0
iface eth0 inet dhcp
```

This means:

* `auto eth0` → bring up `eth0` automatically during boot.
* `iface eth0 inet dhcp` → configure `eth0` using DHCP.

Or a static configuration:

```text
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
```

Here you're manually assigning:

```text
IP       → 192.168.1.100
Netmask  → 255.255.255.0
Gateway  → 192.168.1.1
```

### What is `eth0`?

`eth0` is simply the **name of a network interface**. Modern Linux systems may instead use names such as:

```text
ens33
enp0s3
enp0s8
```

You can see your interfaces with:

```bash
ip addr
```

### Important for Kali

On newer Kali Linux installations, networking is often managed by **NetworkManager**, so `/etc/network/interfaces` may be mostly empty or may not be the main configuration source.

You can check what is managing your network with:

```bash
systemctl status NetworkManager
```

So, in simple terms:

> **`/etc/network/interfaces` = a configuration file that tells Linux how to bring up and configure its network interfaces.**
