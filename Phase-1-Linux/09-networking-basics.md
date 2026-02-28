# Module 9: Networking Basics

## 🎯 Why This Matters in Production

**Scenario 1**: Your app is deployed but nobody can reach it. Firewall rule? Wrong port? Service not listening? DNS broken?

**Scenario 2**: Two microservices can't talk to each other. Same server, different servers, different VPCs — you need to systematically eliminate every layer.

**Scenario 3**: "The website is slow." Is it the app, the network, packet loss, DNS latency, or a saturated link?

**Networking = the plumbing. You don't think about it until it breaks, and when it breaks, everything breaks.**

---

## 🧠 **THE NETWORKING STACK**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    OSI / TCP-IP MODEL                               │
├───────┬──────────────┬──────────────────────────────────────────── │
│ Layer │ Name         │ What it does              │ Tools           │
├───────┼──────────────┼───────────────────────────┼─────────────── │
│  7    │ Application  │ HTTP, DNS, SSH, FTP        │ curl, dig      │
│  4    │ Transport    │ TCP/UDP, ports, connections│ ss, netstat    │
│  3    │ Network      │ IP addresses, routing      │ ip, ping, route│
│  2    │ Data Link    │ MAC addresses, switching   │ ip link, arp   │
│  1    │ Physical     │ Cables, signals            │ ethtool        │
└───────┴──────────────┴───────────────────────────┴─────────────── │
└─────────────────────────────────────────────────────────────────────┘
```

**Troubleshooting strategy: always work bottom-up.**
Can I ping it? → Can I reach the port? → Can I get a response? → Is DNS resolving?

---

## 🌐 **IP ADDRESSES & INTERFACES**

### **ip — The Modern Network Tool**

`ip` replaced the old `ifconfig`. Always use `ip` in modern Linux.

```bash
# Show all network interfaces and their IPs
ip addr
ip a              # short form

# Show a specific interface
ip addr show eth0
ip addr show ens33

# Show only IPv4 addresses
ip -4 addr

# Show only IPv6 addresses
ip -6 addr
```

**Reading `ip addr` output:**

```
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether 00:0c:29:ab:cd:ef brd ff:ff:ff:ff:ff:ff
    inet 192.168.150.128/24 brd 192.168.150.255 scope global dynamic ens33
    inet6 fe80::20c:29ff:feab:cdef/64 scope link
```

| Field | Meaning |
|-------|---------|
| `ens33` | Interface name |
| `UP,LOWER_UP` | Interface is up and has carrier (cable connected) |
| `mtu 1500` | Max transmission unit (bytes per packet) |
| `00:0c:29:ab:cd:ef` | MAC address |
| `192.168.150.128/24` | IP address with subnet mask |
| `/24` | Subnet mask = 255.255.255.0 (256 addresses) |
| `dynamic` | IP assigned via DHCP |

### **Understanding CIDR Notation**

```
┌───────────────────────────────────────────────────────────────────┐
│  192.168.150.128/24                                               │
│                                                                   │
│  /24 = first 24 bits are network, last 8 bits are hosts           │
│                                                                   │
│  192.168.150 = network part (fixed)                               │
│            .128 = host part (variable, 0-255)                     │
│                                                                   │
│  /24 → 256 addresses, 254 usable hosts                           │
│  /25 → 128 addresses, 126 usable hosts                           │
│  /16 → 65536 addresses, 65534 usable hosts                       │
│  /32 → 1 address (single host, used for loopback/VPN)            │
└───────────────────────────────────────────────────────────────────┘
```

### **Interface Management**

```bash
# Bring interface up/down
sudo ip link set ens33 up
sudo ip link set ens33 down

# Assign an IP temporarily (lost on reboot)
sudo ip addr add 192.168.1.100/24 dev ens33

# Remove an IP
sudo ip addr del 192.168.1.100/24 dev ens33

# Show link-layer info (MAC, MTU, state)
ip link show
ip link show ens33
```

### **lo — The Loopback Interface**

```
lo: <LOOPBACK,UP,LOWER_UP>
    inet 127.0.0.1/8
```

- `127.0.0.1` is always "this machine"
- Used by services to talk to themselves
- Always up, even with no network

---

## 🗺️ **ROUTING**

### **ip route — Show and Modify Routing Table**

```bash
# Show routing table
ip route
ip r              # short form

# Example output:
# default via 192.168.150.2 dev ens33 proto dhcp
# 192.168.150.0/24 dev ens33 proto kernel scope link src 192.168.150.128
```

**Reading routes:**
| Route | Meaning |
|-------|---------|
| `default via 192.168.150.2` | All traffic goes to gateway 192.168.150.2 (your router) |
| `192.168.150.0/24 dev ens33` | Traffic for this subnet goes directly via ens33 |

```bash
# Check what route a packet to an IP would take
ip route get 8.8.8.8
# Output: 8.8.8.8 via 192.168.150.2 dev ens33 src 192.168.150.128

# Add a static route
sudo ip route add 10.0.0.0/8 via 192.168.1.1

# Delete a route
sudo ip route del 10.0.0.0/8

# Show default gateway
ip route | grep default
```

---

## 📡 **CONNECTIVITY TESTING**

### **ping — Test Reachability**

`ping` sends ICMP echo requests and measures response time.

```bash
# Basic ping
ping google.com

# Ping with count
ping -c 4 google.com

# Ping with interval
ping -i 0.5 google.com     # Every 0.5 seconds

# Ping with specific size
ping -s 1000 google.com    # 1000 byte packets

# Flood ping (must be root)
sudo ping -f 192.168.1.1

# Set TTL
ping -t 10 google.com
```

**Reading ping output:**

```
PING google.com (142.250.80.14): 56 bytes
64 bytes from 142.250.80.14: icmp_seq=0 ttl=115 time=12.5 ms
64 bytes from 142.250.80.14: icmp_seq=1 ttl=115 time=11.8 ms

--- google.com ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 11.8/12.1/12.5/0.3 ms
```

| Field | Meaning |
|-------|---------|
| `ttl=115` | Time-to-live (hops remaining). Linux starts at 64, Windows at 128 |
| `time=12.5ms` | Round-trip latency |
| `0% packet loss` | All packets delivered |
| `mdev` | Mean deviation (latency jitter — high jitter = unstable connection) |

**What ping results tell you:**

```bash
# ping succeeds → Layer 3 (IP) works
# ping fails → Could be: host down, firewall blocking ICMP, or wrong IP

# ping local gateway
ping -c 3 $(ip route | awk '/default/ {print $3}')

# ping loopback (always should work)
ping -c 1 127.0.0.1

# Test without DNS (use IP directly)
ping -c 1 8.8.8.8
```

---

### **traceroute / tracepath — Trace Network Path**

```bash
# Install if needed
sudo apt install traceroute

# Trace route to a host
traceroute google.com

# Use ICMP (instead of UDP)
traceroute -I google.com

# Tracepath (no root needed)
tracepath google.com
```

**Reading traceroute:**

```
 1  192.168.150.2 (192.168.150.2)    1.2 ms    # Your router
 2  10.0.0.1 (10.0.0.1)             5.4 ms    # ISP
 3  * * *                                      # Router not responding to probes
 4  72.14.210.85                    12.1 ms    # Google network
 5  142.250.80.14                   12.5 ms    # Google server
```

- `* * *` = that router doesn't respond to traceroute (not necessarily broken)
- Latency jumps between hops show where slowness is introduced

---

## 🔌 **PORTS & SOCKETS**

### **Understanding Ports**

```
┌─────────────────────────────────────────────────────────────────────┐
│  Port = a numbered "door" on a host                                 │
│                                                                     │
│  IP address = which building (host)                                 │
│  Port = which apartment (service)                                   │
│                                                                     │
│  Full address: 192.168.1.10:80                                      │
│                      IP    Port                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Common well-known ports:**

| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 25 | TCP | SMTP (email) |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 8080 | TCP | HTTP alt / Tomcat |
| 27017 | TCP | MongoDB |

### **ss — Socket Statistics (Modern netstat)**

`ss` shows all open sockets. It replaced `netstat`.

```bash
# All listening ports
ss -tlnp

# All connections (listening + established)
ss -tanp

# Options breakdown:
# -t  TCP sockets
# -u  UDP sockets
# -l  Listening only
# -n  No hostname resolution (faster, shows IPs)
# -p  Show process using the socket
# -a  All (listening + connected)
```

**Reading `ss -tlnp` output:**

```
State   Recv-Q Send-Q Local Address:Port  Peer Address:Port  Process
LISTEN  0      128    0.0.0.0:22          0.0.0.0:*          users:(("sshd",pid=1234))
LISTEN  0      511    127.0.0.1:5432      0.0.0.0:*          users:(("postgres",pid=5678))
LISTEN  0      128    *:80                *:*                 users:(("nginx",pid=910))
```

| Field | Meaning |
|-------|---------|
| `0.0.0.0:22` | Listening on ALL interfaces on port 22 |
| `127.0.0.1:5432` | Listening on LOCALHOST only (not reachable externally) |
| `*:80` | Listening on all IPv4 and IPv6 |

```bash
# Is something listening on port 80?
ss -tlnp | grep :80

# What process is using port 8080?
ss -tlnp | grep :8080

# All established connections
ss -tnp | grep ESTABLISHED

# Count established connections
ss -tn | grep ESTABLISHED | wc -l

# Show UDP sockets
ss -ulnp
```

---

### **netstat — Old but Still Used**

```bash
# Install if needed
sudo apt install net-tools

# Show listening ports (same as ss -tlnp)
netstat -tlnp

# All connections
netstat -tanp

# Show routing table
netstat -r
```

> Use `ss` over `netstat` — it's faster and ships with modern Linux by default.

---

## 🌍 **DNS — Domain Name System**

DNS translates human names to IP addresses. It's involved in nearly every network connection.

```
┌─────────────────────────────────────────────────────────────────────┐
│  DNS RESOLUTION FLOW                                                │
│                                                                     │
│  You type: google.com                                               │
│       ↓                                                             │
│  1. Check local cache                                               │
│       ↓ (not found)                                                 │
│  2. Check /etc/hosts                                                │
│       ↓ (not found)                                                 │
│  3. Ask DNS resolver (from /etc/resolv.conf)                        │
│       ↓ (e.g. 8.8.8.8)                                             │
│  4. Recursive resolution → returns 142.250.80.14                   │
│       ↓                                                             │
│  5. Your machine connects to 142.250.80.14                          │
└─────────────────────────────────────────────────────────────────────┘
```

### **dig — DNS Query Tool**

```bash
# Basic query (A record = IPv4 address)
dig google.com

# Just the answer (clean output)
dig google.com +short

# Query a specific DNS server
dig @8.8.8.8 google.com

# Look up different record types
dig google.com A        # IPv4 address
dig google.com AAAA     # IPv6 address
dig google.com MX       # Mail servers
dig google.com NS       # Name servers
dig google.com TXT      # Text records (SPF, DKIM, etc.)
dig google.com CNAME    # Canonical name (alias)

# Reverse DNS lookup (IP → name)
dig -x 8.8.8.8

# Trace the full resolution path
dig google.com +trace
```

**Reading `dig` output:**

```
;; ANSWER SECTION:
google.com.    299    IN    A    142.250.80.14
               ^TTL        ^Type ^IP

;; Query time: 12 msec
;; SERVER: 192.168.150.2#53
```

| Field | Meaning |
|-------|---------|
| `299` | TTL in seconds (cache expires in 5 minutes) |
| `IN A` | Internet class, A record type |
| `SERVER: 192.168.150.2#53` | Which DNS server answered |

### **nslookup — Simpler DNS Query**

```bash
# Basic lookup
nslookup google.com

# Use specific DNS server
nslookup google.com 8.8.8.8

# Reverse lookup
nslookup 8.8.8.8
```

### **DNS Configuration Files**

```bash
# Your DNS resolver config
cat /etc/resolv.conf
# nameserver 192.168.150.2
# nameserver 8.8.8.8

# Local hostname overrides (highest priority!)
cat /etc/hosts
# 127.0.0.1    localhost
# 127.0.1.1    sam-VMware-Virtual-Platform
# 192.168.1.50 myapp.internal

# DNS resolution order
cat /etc/nsswitch.conf | grep hosts
# hosts: files dns   → /etc/hosts first, then DNS
```

```bash
# Add a local hostname override (useful for development)
echo "192.168.1.50  myapp.internal" | sudo tee -a /etc/hosts

# Flush DNS cache (systemd-resolved)
sudo resolvectl flush-caches

# Check DNS stats
resolvectl statistics

# Show current DNS server being used
resolvectl status
```

---

## 🔗 **HTTP TESTING**

### **curl — Transfer Data to/from URLs**

`curl` is the Swiss Army knife for HTTP — testing APIs, downloading files, checking connectivity.

```bash
# Basic GET request
curl https://google.com

# Follow redirects
curl -L https://google.com

# Show only headers
curl -I https://google.com

# Save output to file
curl -o page.html https://google.com
curl -O https://example.com/file.zip   # Save with original filename

# Send POST request
curl -X POST -d '{"key":"value"}' -H "Content-Type: application/json" https://api.example.com

# With authentication
curl -u username:password https://api.example.com
curl -H "Authorization: Bearer TOKEN" https://api.example.com

# Show verbose output (headers, SSL, timing)
curl -v https://google.com

# Just the HTTP status code
curl -s -o /dev/null -w "%{http_code}" https://google.com

# Response time breakdown
curl -s -o /dev/null -w "DNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTotal: %{time_total}s\n" https://google.com

# Follow all redirects, show final URL
curl -Ls -o /dev/null -w "%{url_effective}" https://bit.ly/something
```

### **wget — Download Files**

```bash
# Download a file
wget https://example.com/file.tar.gz

# Download quietly (no progress bar)
wget -q https://example.com/file.tar.gz

# Download to specific location
wget -O /tmp/file.tar.gz https://example.com/file.tar.gz

# Download with retry
wget --tries=3 https://example.com/file.tar.gz

# Test URL (don't download, just check)
wget --spider https://example.com/page
```

---

## 🔒 **SSH — Secure Shell**

SSH is how you connect to remote servers. In production, you'll live in SSH.

### **Basic SSH**

```bash
# Connect to a server
ssh username@hostname
ssh sam@192.168.1.50
ssh sam@myserver.example.com

# Connect on non-standard port
ssh -p 2222 sam@192.168.1.50

# Connect with a private key
ssh -i ~/.ssh/my_key.pem sam@192.168.1.50

# Run a command remotely
ssh sam@192.168.1.50 'uptime && free -h'

# Copy files to remote (secure copy)
scp file.txt sam@192.168.1.50:/home/sam/
scp -r /local/dir sam@192.168.1.50:/remote/dir

# Copy files from remote
scp sam@192.168.1.50:/var/log/app.log /tmp/

# Sync directories (rsync over SSH)
rsync -avz /local/dir sam@192.168.1.50:/remote/dir
```

### **SSH Keys — How They Work**

```
┌─────────────────────────────────────────────────────────────────────┐
│  SSH KEY AUTHENTICATION                                             │
│                                                                     │
│  Your machine                      Remote server                   │
│  ┌──────────────┐                 ┌──────────────────┐             │
│  │ Private key  │  ←── NEVER      │  ~/.ssh/         │             │
│  │ ~/.ssh/id_rsa│     share       │  authorized_keys │             │
│  │              │                 │  (public key)    │             │
│  │ Public key   │ ─── copy to ──▶ │                  │             │
│  │ id_rsa.pub   │                 │                  │             │
│  └──────────────┘                 └──────────────────┘             │
│                                                                     │
│  At login: server sends challenge, only your private key can        │
│  respond correctly → authentication without password               │
└─────────────────────────────────────────────────────────────────────┘
```

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -C "sam@work"
# Creates: ~/.ssh/id_ed25519 (private) and ~/.ssh/id_ed25519.pub (public)

# Copy public key to remote server
ssh-copy-id sam@192.168.1.50

# Manually add public key on server
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys

# Test key-based login
ssh -i ~/.ssh/id_ed25519 sam@192.168.1.50

# Permissions MUST be correct (SSH is strict)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys
```

### **SSH Config File — Quality of Life**

```bash
# ~/.ssh/config
Host myserver
    HostName 192.168.1.50
    User sam
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host prod-web
    HostName 10.0.1.100
    User ubuntu
    Port 2222
    IdentityFile ~/.ssh/prod_key.pem

# Now you can just do:
ssh myserver
ssh prod-web
```

---

## 🔍 **PORT TESTING**

### **nc (netcat) — Network Swiss Army Knife**

```bash
# Test if a port is open
nc -zv 192.168.1.50 22
nc -zv google.com 443

# -z = zero-I/O mode (just test connection)
# -v = verbose

# Test a range of ports
nc -zv 192.168.1.50 20-25

# Listen on a port (simple server for testing)
nc -l 8080

# Connect to a listening server
nc 192.168.1.50 8080

# Simple file transfer
# On receiver:
nc -l 9999 > received_file.txt
# On sender:
nc 192.168.1.50 9999 < file_to_send.txt
```

### **telnet — Old Port Tester**

```bash
# Test if port is open
telnet 192.168.1.50 22
telnet google.com 80

# Exit: Ctrl+] then type quit
```

### **nmap — Network Scanner**

```bash
# Install
sudo apt install nmap

# Scan a single host
nmap 192.168.1.50

# Scan specific ports
nmap -p 22,80,443 192.168.1.50

# Scan a subnet
nmap 192.168.1.0/24

# Fast scan (most common ports)
nmap -F 192.168.1.50

# Detect OS and service versions
nmap -A 192.168.1.50

# Scan without ping (for hosts blocking ICMP)
nmap -Pn 192.168.1.50
```

> ⚠️ Only scan networks you own or have permission to scan. Unauthorized scanning is illegal.

---

## 🧱 **FIREWALL — ufw**

Ubuntu uses `ufw` (Uncomplicated Firewall) as a frontend for `iptables`.

```bash
# Check firewall status
sudo ufw status
sudo ufw status verbose

# Enable/disable firewall
sudo ufw enable
sudo ufw disable

# Allow a port
sudo ufw allow 22           # SSH
sudo ufw allow 80           # HTTP
sudo ufw allow 443          # HTTPS
sudo ufw allow 8080/tcp     # Specific protocol

# Allow from specific IP
sudo ufw allow from 192.168.1.100
sudo ufw allow from 192.168.1.100 to any port 22

# Deny a port
sudo ufw deny 23            # Telnet

# Delete a rule
sudo ufw delete allow 8080
sudo ufw delete deny 23

# Allow an app profile
sudo ufw app list           # List available profiles
sudo ufw allow 'Nginx Full' # Allow HTTP + HTTPS for nginx

# View numbered rules
sudo ufw status numbered

# Reset firewall
sudo ufw reset
```

**Golden rule**: Always allow SSH **before** enabling the firewall, or you'll lock yourself out! 😱

```bash
# Safe sequence when setting up firewall on a server
sudo ufw allow 22
sudo ufw enable
sudo ufw status
```

---

## 🏭 **PRODUCTION NETWORKING WORKFLOWS**

### **Workflow 1: Can't Connect to a Service**

```bash
# Step 1: Is the service running and listening?
ss -tlnp | grep :80

# Step 2: Can I reach it from localhost?
curl -I http://localhost:80
nc -zv localhost 80

# Step 3: Is there a firewall blocking?
sudo ufw status

# Step 4: Can I reach it from another machine?
# (From another machine:)
nc -zv 192.168.1.50 80
curl http://192.168.1.50:80

# Step 5: Is DNS resolving correctly?
dig myapp.example.com
# vs
curl http://192.168.1.50:80    # Try direct IP to bypass DNS
```

### **Workflow 2: Slow Network / High Latency**

```bash
# Step 1: Baseline latency
ping -c 10 google.com
# Look at avg and mdev (jitter)

# Step 2: Where does it get slow?
traceroute google.com

# Step 3: Is it DNS slow?
time dig google.com
# If DNS takes > 100ms, your resolver is slow

# Step 4: Is it connection time or content download?
curl -s -o /dev/null -w "DNS: %{time_namelookup}s Connect: %{time_connect}s Total: %{time_total}s\n" https://google.com
```

### **Workflow 3: Two Services Can't Talk to Each Other**

```bash
# Step 1: Can service A ping service B?
ping -c 3 service-b-ip

# Step 2: Is service B listening?
ss -tlnp | grep :PORT     # Run on service B's machine

# Step 3: Can A reach B's port?
nc -zv service-b-ip PORT  # Run from service A's machine

# Step 4: Check firewall on B
sudo ufw status           # On service B

# Step 5: Check /etc/hosts or DNS
dig service-b-hostname    # Check name resolves to right IP
```

### **Workflow 4: Finding What's Using a Port**

```bash
# What's on port 8080?
ss -tlnp | grep :8080

# More detail
sudo ss -tlnp | grep :8080
# Output: users:(("java",pid=12345,fd=45))

# Or via process
sudo lsof -i :8080

# Kill it if needed
sudo kill 12345
```

---

## ⚠️ **COMMON MISTAKES**

### **1. Firewall Before SSH Rule**

```bash
# WRONG order — you're now locked out
sudo ufw enable              # Locks you out if SSH not allowed!
sudo ufw allow 22            # Too late

# RIGHT order
sudo ufw allow 22            # Allow SSH first
sudo ufw enable              # THEN enable
```

### **2. Service Listening Only on Localhost**

```bash
# Service works from localhost but not from outside:
ss -tlnp | grep :5432
# LISTEN 0 128 127.0.0.1:5432    ← only localhost!

# Need to change config to listen on 0.0.0.0 or specific interface
# In postgresql.conf: listen_addresses = '*'
```

### **3. Testing with Hostname when DNS is Broken**

```bash
# "I can't reach myapp.example.com"
ping myapp.example.com
# FAIL — is it the app or DNS?

# TEST with direct IP
ping 192.168.1.50
# If this works → DNS broken, not the app
```

### **4. Forgetting Port in URLs**

```bash
# App listens on 8080 but you test on 80
curl http://localhost           # Tests port 80 — wrong!
curl http://localhost:8080      # Correct
```

### **5. SSH Permission Errors**

```bash
# "Permission denied (publickey)"
# Usually a file permission problem

ls -la ~/.ssh/
# Should be:
# drwx------  ~/.ssh/
# -rw-------  authorized_keys
# -rw-------  id_ed25519
# -rw-r--r--  id_ed25519.pub

chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/id_ed25519
```

---

## 📝 **QUICK REFERENCE**

```bash
# ============== INTERFACES ==============
ip addr                         # Show all interfaces & IPs
ip link show                    # Show interface states
ip route                        # Show routing table
ip route get 8.8.8.8            # Show route for specific IP

# ============== CONNECTIVITY ==============
ping -c 4 google.com            # Test reachability
traceroute google.com           # Trace network path
nc -zv host port                # Test if port is open

# ============== PORTS & SOCKETS ==============
ss -tlnp                        # Listening TCP ports
ss -tanp                        # All TCP connections
ss -ulnp                        # Listening UDP ports
ss -tlnp | grep :80             # Who's on port 80?
sudo lsof -i :8080              # Process on port 8080

# ============== DNS ==============
dig hostname                    # DNS lookup
dig hostname +short             # Just the IP
dig @8.8.8.8 hostname           # Query specific DNS
dig -x 8.8.8.8                  # Reverse lookup
cat /etc/resolv.conf            # DNS server config
cat /etc/hosts                  # Local host overrides

# ============== HTTP ==============
curl -I https://url             # Headers only
curl -L https://url             # Follow redirects
curl -s -o /dev/null -w "%{http_code}" url  # Just status code
wget https://url/file           # Download file

# ============== SSH ==============
ssh user@host                   # Connect
ssh -i key.pem user@host        # With key
scp file user@host:/path        # Copy to remote
ssh-keygen -t ed25519           # Generate key pair
ssh-copy-id user@host           # Copy public key

# ============== FIREWALL ==============
sudo ufw status                 # Check firewall
sudo ufw allow 22               # Allow SSH
sudo ufw allow 80/tcp           # Allow HTTP
sudo ufw deny 23                # Deny telnet
sudo ufw delete allow 8080      # Remove rule
```

---

## 🎯 **MINI-TASK: Network Investigation**

### **Part 1: Interface & Routing**

1. **Your interfaces**: Run `ip addr` — list all interfaces, their IPs, and state (UP/DOWN)

2. **Your routes**: Run `ip route` — what is your default gateway IP?

3. **Route decision**: Run `ip route get 8.8.8.8` — what interface and gateway will traffic to Google use?

4. **MAC address**: What is the MAC address of your main interface?

### **Part 2: Connectivity**

5. **Ping your gateway**: Ping your default gateway 4 times — what's the avg latency?

6. **Ping Google**: `ping -c 4 8.8.8.8` — any packet loss?

7. **Ping by name**: `ping -c 2 google.com` — does DNS work? Compare latency to task 6.

8. **Trace a path**: Run `traceroute -n google.com` (or `tracepath google.com`) — how many hops to reach Google?

### **Part 3: Ports & Services**

9. **Listening services**: Run `ss -tlnp` — list every port your machine is listening on and which service owns it.

10. **SSH service**: Is SSH (`sshd`) listening? On which IP:port? Is it `0.0.0.0:22` or `127.0.0.1:22`? What's the difference?

11. **Established connections**: Run `ss -tnp | grep ESTABLISHED` — what active connections exist right now?

12. **Test a port**: Use `nc -zv localhost 22` — does it succeed?

### **Part 4: DNS**

13. **Basic lookup**: `dig google.com +short` — what IP(s) does google.com resolve to?

14. **Your DNS server**: `cat /etc/resolv.conf` — what DNS server is your machine using?

15. **TTL check**: `dig google.com` — what is the TTL of the A record? What does that number mean?

16. **Reverse lookup**: `dig -x 8.8.8.8 +short` — what hostname does Google's DNS server resolve to?

17. **Your hosts file**: `cat /etc/hosts` — list any entries beyond `127.0.0.1 localhost`

### **Part 5: Firewall**

18. **Firewall status**: `sudo ufw status` — is it enabled? What rules exist?

19. **Allow and verify SSH rule** (if not already there):
    ```bash
    sudo ufw allow 22
    sudo ufw status
    ```

### **Security Questions:**

1. What's the difference between `0.0.0.0:22` and `127.0.0.1:22` in `ss` output?
2. You can ping a server's IP but `curl http://myapp.example.com` fails — what's the likely cause?
3. Why must you `allow 22` BEFORE `ufw enable`?
4. What is TTL in a DNS response and why does it matter?
5. What's the difference between `ssh -i key.pem` and password-based SSH, and why do production servers use keys?

---

Create `/home/sam/DevopsRoadmap/Phase-1-Linux/networking-report.txt` with all commands and their outputs. Explain your observations!
