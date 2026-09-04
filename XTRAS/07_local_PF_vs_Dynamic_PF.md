**Local port forwarding** and **dynamic port forwarding** are two common ways to create SSH tunnels. Both let you securely route traffic through an SSH connection, but they work differently.

### Local Port Forwarding (`-L`)

This forwards a **specific local port** on your machine to a **specific remote host:port**, using the SSH server as a jump point.

**Command syntax:**
```bash
ssh -L [local_bind_address:]local_port:destination_host:destination_port user@ssh_server
```

**How it works:**
1. You connect to the SSH server.
2. Anything that connects to `localhost:local_port` on your machine is sent through the encrypted SSH tunnel to the SSH server.
3. The SSH server then connects to `destination_host:destination_port` and relays the traffic.

**Common use cases:**
- Accessing a service that is only reachable from the remote network (e.g., a database, internal web app, or RDP server).
- Example: Access a remote MySQL server that only listens on the internal network:
  ```bash
  ssh -L 3306:db.internal:3306 user@bastion.example.com
  ```
  Then connect your local MySQL client to `localhost:3306`.

- Accessing a service running on the SSH server itself:
  ```bash
  ssh -L 8080:localhost:80 user@server
  ```
  Now `http://localhost:8080` shows the web server running on the remote host.

**Key points:**
- Destination is fixed when you create the tunnel.
- You must specify both the local port and the exact remote destination.
- Works well for single services.

### Dynamic Port Forwarding (`-D`)

This creates a **SOCKS proxy** on your local machine. Applications that support SOCKS can then send *any* traffic through the tunnel, and the SSH server acts as the exit point.

**Command syntax:**
```bash
ssh -D [local_bind_address:]local_port user@ssh_server
```

**How it works:**
1. SSH opens a SOCKS proxy (usually SOCKS5) listening on `localhost:local_port`.
2. Any application configured to use that SOCKS proxy sends its traffic through the SSH tunnel.
3. The SSH server makes the actual connections to whatever destinations the application requests.

**Common use cases:**
- Browsing the web as if you were on the remote network (useful for accessing geo-restricted or internal sites).
- Tunneling traffic from applications that support SOCKS proxies (browsers, curl, some tools, etc.).
- Example:
  ```bash
  ssh -D 1080 user@bastion.example.com
  ```
  Then configure your browser (or system) to use SOCKS5 proxy `127.0.0.1:1080`.

**Key points:**
- Destination is **not** fixed — any host/port the client requests can be reached.
- More flexible than local forwarding.
- Requires the application to support SOCKS proxies (or you can use tools like `proxychains` / `tsocks`).

### Quick Comparison

| Feature                  | Local Port Forwarding (`-L`)          | Dynamic Port Forwarding (`-D`)      |
|--------------------------|---------------------------------------|-------------------------------------|
| Type of tunnel           | Fixed destination                     | SOCKS proxy (flexible destinations) |
| Number of destinations   | One specific host:port                | Any host:port the client requests   |
| Application support      | Any application (connect to localhost)| Applications that support SOCKS     |
| Typical use              | Access one internal service           | General-purpose proxy / browsing    |
| Command flag             | `-L`                                  | `-D`                                |

### Useful Related Options

- `-N` — Do not execute a remote command (useful when you only want the tunnel).
- `-f` — Background the SSH process after authentication.
- `-C` — Enable compression (can help with slow links).
- Bind address: By default the local port listens only on localhost. Use `0.0.0.0:port` (or just `*:port` in some versions) if you want other machines on your network to use the tunnel (be careful with security).

**Examples with background + no remote shell:**
```bash
# Local forwarding
ssh -f -N -L 3306:db.internal:3306 user@bastion

# Dynamic (SOCKS) forwarding
ssh -f -N -D 1080 user@bastion
```

### Security Notes
- Only forward ports you need.
- Prefer binding to `127.0.0.1` unless you deliberately want the tunnel accessible from other hosts.
- Dynamic forwarding effectively gives the remote server the ability to make outbound connections on your behalf, so use trusted SSH servers.

