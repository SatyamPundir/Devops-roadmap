# Module 4: System Services (systemd)

## 🎯 Why This Matters in Production

**Scenario**: It's 3 AM. Your monitoring system alerts: "Flask app is down!" You SSH into the server.

You need to instantly know:
- **How do I check if the service is running?**
- **How do I restart it?**
- **How do I make sure it starts automatically after reboot?**
- **How do I see why it crashed?**

**In production, systemd is how you manage EVERYTHING that runs on your server.**

---

## 🧠 **WHAT IS systemd? (THE FOUNDATION)**

### **Definition**

**systemd** is the **init system and service manager** for modern Linux. It's PID 1 — the first process that starts when Linux boots.

```
BIOS/UEFI → Bootloader → Kernel → systemd (PID 1) → Everything else
                                      │
                                      ├─ Mounts filesystems
                                      ├─ Starts networking
                                      ├─ Starts your services
                                      ├─ Manages dependencies
                                      └─ Monitors everything
```

**Think of it like:**
- **systemd** = Orchestra conductor
- **Services** = Musicians
- **Unit files** = Sheet music (instructions)

---

### **Why systemd Replaced SysVinit**

| Feature | Old (SysVinit) | New (systemd) |
|---------|----------------|---------------|
| Boot speed | Sequential (slow) | Parallel (fast) |
| Service files | Shell scripts | Declarative unit files |
| Dependency handling | Manual | Automatic |
| Process monitoring | None | Built-in (restart on crash) |
| Logging | Scattered text files | Centralized journal |
| Cgroups support | None | Native |

---

### **Key Concepts**

**1. Units** — Everything systemd manages
```
Types of units:
├─ .service  → Daemons/programs (nginx, postgresql)
├─ .socket   → Network sockets
├─ .timer    → Scheduled tasks (like cron)
├─ .mount    → Filesystem mounts
├─ .target   → Groups of units (like runlevels)
└─ .path     → File/directory monitoring
```

**2. Unit Files** — Configuration files that describe units
```
Location hierarchy (priority order):
/etc/systemd/system/      ← Admin customizations (highest priority)
/run/systemd/system/      ← Runtime units
/lib/systemd/system/      ← Package defaults (lowest priority)
```

**3. Targets** — Groups of units (like old runlevels)
```
multi-user.target  ← Normal server boot (no GUI)
graphical.target   ← Desktop with GUI
rescue.target      ← Single-user recovery mode
```

---

### 🎯 **Running Example: Web Application Stack**

Throughout this module, we'll manage this **production stack**:

```
┌─────────────────────────────────────────┐
│         NGINX Web Server                │
│         Service: nginx.service          │
│         Port: 80, 443                   │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│      Flask Application (Gunicorn)       │
│      Service: flask-app.service         │
│      Port: 5000                         │
└─────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│      PostgreSQL Database                │
│      Service: postgresql.service        │
│      Port: 5432                         │
└─────────────────────────────────────────┘

Dependency chain:
postgresql → flask-app → nginx
(DB must start first, then app, then web server)
```

---

## ⚡ **systemctl - THE MAIN COMMAND**

`systemctl` is your primary tool for managing systemd services.

### **Basic Syntax**

```bash
systemctl [COMMAND] [UNIT]

# Examples:
systemctl status nginx
systemctl start nginx
systemctl stop nginx
```

---

### **Essential Commands**

| Command | What It Does | When To Use |
|---------|--------------|-------------|
| `status` | Show service state & recent logs | First thing to check |
| `start` | Start a service | After deploy/maintenance |
| `stop` | Stop a service | Before maintenance |
| `restart` | Stop then start | After config changes |
| `reload` | Reload config without restart | Zero-downtime config update |
| `enable` | Start on boot | New service setup |
| `disable` | Don't start on boot | Decommissioning |
| `is-active` | Check if running (for scripts) | Automation |
| `is-enabled` | Check if starts on boot | Auditing |

---

## 🔍 **CHECKING SERVICE STATUS**

### **The status Command**

```bash
$ sudo systemctl status nginx
```

**Output breakdown:**

```
● nginx.service - A high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-01-23 10:00:00 UTC; 2h ago
       Docs: man:nginx(8)
    Process: 1234 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 1235 (nginx)
      Tasks: 5 (limit: 4915)
     Memory: 12.5M
        CPU: 1.234s
     CGroup: /system.slice/nginx.service
             ├─1235 "nginx: master process /usr/sbin/nginx"
             ├─1236 "nginx: worker process"
             └─1237 "nginx: worker process"

Jan 23 10:00:00 server nginx[1234]: Starting nginx...
Jan 23 10:00:00 server nginx[1235]: nginx started successfully
```

**Understanding each field:**

| Field | Meaning |
|-------|---------|
| `●` (green dot) | Service is running (red = stopped/failed) |
| `Loaded` | Unit file path, enabled/disabled status |
| `Active` | Current state and how long it's been running |
| `Main PID` | Process ID (matches what you'd see in `ps`) |
| `Tasks` | Number of threads/processes |
| `Memory` | RAM usage |
| `CGroup` | Process tree within control group |
| Bottom logs | Recent journal entries |

---

### **Service States**

| State | Symbol | Meaning |
|-------|--------|---------|
| `active (running)` | ● (green) | Service is running normally |
| `active (exited)` | ● (green) | One-shot service completed successfully |
| `inactive (dead)` | ○ (white) | Service is stopped |
| `failed` | ● (red) | Service crashed or failed to start |
| `activating` | ● (yellow) | Service is starting up |
| `deactivating` | ● (yellow) | Service is shutting down |

---

### **Quick Status Checks (for Scripts)**

```bash
# Check if running (returns 0 or 1)
$ systemctl is-active nginx
active

$ systemctl is-active stopped-service
inactive

# Use in scripts
if systemctl is-active --quiet nginx; then
    echo "Nginx is running"
else
    echo "Nginx is NOT running!"
fi

# Check if enabled for boot
$ systemctl is-enabled nginx
enabled

$ systemctl is-enabled some-service
disabled
```

---

## 🚀 **STARTING AND STOPPING SERVICES**

### **start - Start a Service**

```bash
$ sudo systemctl start nginx

# No output = success
# Check status to confirm
$ systemctl status nginx
● nginx.service - A high performance web server
     Active: active (running) since...
```

**What happens internally:**
1. systemd reads `/lib/systemd/system/nginx.service`
2. Checks dependencies (are required services running?)
3. Runs `ExecStart` command
4. Monitors the process

---

### **stop - Stop a Service**

```bash
$ sudo systemctl stop nginx

# Check it stopped
$ systemctl status nginx
○ nginx.service - A high performance web server
     Active: inactive (dead) since...
```

**What happens:**
1. systemd sends SIGTERM to main process
2. Waits for graceful shutdown (TimeoutStopSec)
3. If still running, sends SIGKILL

---

### **restart vs reload**

**restart** — Full stop and start:
```bash
$ sudo systemctl restart nginx

# What happens:
1. Stop nginx (SIGTERM → wait → SIGKILL if needed)
2. Start nginx fresh
3. Brief downtime during restart!
```

**reload** — Reload config without stopping:
```bash
$ sudo systemctl reload nginx

# What happens:
1. Send SIGHUP to nginx
2. Nginx reads new config
3. Gracefully switch to new config
4. ZERO downtime!
```

**When to use which:**

| Scenario | Use |
|----------|-----|
| Changed config file | `reload` (if supported) |
| Updated binary/package | `restart` |
| Service is misbehaving | `restart` |
| Not sure if reload works | `reload-or-restart` |

```bash
# Safe option: reload if possible, restart if not
$ sudo systemctl reload-or-restart nginx
```

---

### **Real-World Example: Nginx Config Change**

**Scenario:** You updated `/etc/nginx/nginx.conf`

```bash
# Step 1: Test config syntax FIRST
$ sudo nginx -t
nginx: configuration file /etc/nginx/nginx.conf test is successful

# Step 2: Reload (zero downtime)
$ sudo systemctl reload nginx

# Step 3: Verify
$ systemctl status nginx
● nginx.service - A high performance web server
     Active: active (running)...
```

**BEFORE (old config):**
```
Requests → nginx (old config) → flask-app
           └─ No interruption during reload
```

**AFTER (new config applied):**
```
Requests → nginx (NEW config) → flask-app
           └─ Seamless transition
```

---

## 🔄 **ENABLING SERVICES (Boot Persistence)**

### **The Problem**

```bash
# You start nginx
$ sudo systemctl start nginx
# Works great!

# Server reboots...
$ sudo reboot

# After reboot
$ systemctl status nginx
○ nginx.service - inactive (dead)
# nginx didn't start! Users can't reach your site!
```

---

### **enable - Start on Boot**

```bash
$ sudo systemctl enable nginx
Created symlink /etc/systemd/system/multi-user.target.wants/nginx.service → /lib/systemd/system/nginx.service

# Now nginx will start automatically on boot
```

**What `enable` actually does:**
```
Creates a symlink in the target's "wants" directory:

/etc/systemd/system/multi-user.target.wants/
└── nginx.service → /lib/systemd/system/nginx.service

When system reaches multi-user.target, it starts all "wanted" services.
```

---

### **enable vs start**

| Command | Starts Now? | Starts on Boot? |
|---------|-------------|-----------------|
| `start` | ✅ Yes | ❌ No |
| `enable` | ❌ No | ✅ Yes |
| `enable --now` | ✅ Yes | ✅ Yes |

```bash
# Common mistake: only enable, forget to start
$ sudo systemctl enable nginx
$ systemctl status nginx
○ inactive (dead)  ← Not running yet!

# Better: enable AND start in one command
$ sudo systemctl enable --now nginx
$ systemctl status nginx
● active (running)  ← Running and will start on boot
```

---

### **disable - Remove from Boot**

```bash
$ sudo systemctl disable nginx
Removed /etc/systemd/system/multi-user.target.wants/nginx.service

# Service still running, but won't start on next boot
$ systemctl status nginx
● active (running)  ← Still running NOW

# To stop AND disable:
$ sudo systemctl disable --now nginx
```

---

### **Production Setup Pattern**

When deploying a new service:

```bash
# 1. Start the service
$ sudo systemctl start flask-app

# 2. Verify it's working
$ systemctl status flask-app
$ curl http://localhost:5000/health

# 3. Enable for boot (only after confirming it works!)
$ sudo systemctl enable flask-app

# Or all in one (if confident):
$ sudo systemctl enable --now flask-app
```

---

## 📜 **UNIT FILES - THE CONFIGURATION**

### **Anatomy of a Unit File**

Unit files are INI-style configuration files with three main sections:

```ini
[Unit]
# Metadata and dependencies

[Service]
# How to run the service

[Install]
# How to enable the service
```

---

### **Example: Nginx Unit File**

```bash
$ cat /lib/systemd/system/nginx.service
```

```ini
[Unit]
Description=A high performance web server and reverse proxy
Documentation=man:nginx(8)
After=network.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStartPre=/usr/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/bin/kill -s QUIT $MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

---

### **[Unit] Section - Metadata & Dependencies**

| Directive | Purpose | Example |
|-----------|---------|---------|
| `Description` | Human-readable name | `Description=My Flask App` |
| `Documentation` | Link to docs | `Documentation=https://docs.example.com` |
| `After` | Start AFTER these units | `After=network.target postgresql.service` |
| `Before` | Start BEFORE these units | `Before=nginx.service` |
| `Requires` | Hard dependency (fails if dep fails) | `Requires=postgresql.service` |
| `Wants` | Soft dependency (continues if dep fails) | `Wants=redis.service` |

**Understanding Dependencies:**

```
After vs Requires:

After=postgresql.service
  └─ "Start me AFTER postgresql" (ordering only)
  └─ If postgresql fails to start, I still try to start

Requires=postgresql.service
  └─ "I NEED postgresql" (dependency)
  └─ If postgresql fails, I fail too

Usually use BOTH together:
After=postgresql.service
Requires=postgresql.service
```

---

### **[Service] Section - How to Run**

| Directive | Purpose | Common Values |
|-----------|---------|---------------|
| `Type` | How systemd tracks the process | `simple`, `forking`, `oneshot`, `notify` |
| `ExecStart` | Command to start service | `/usr/bin/python /app/main.py` |
| `ExecStop` | Command to stop (optional) | `/bin/kill -s QUIT $MAINPID` |
| `ExecReload` | Command to reload config | `/bin/kill -s HUP $MAINPID` |
| `Restart` | When to auto-restart | `always`, `on-failure`, `no` |
| `RestartSec` | Delay before restart | `5` (seconds) |
| `User` | Run as this user | `www-data` |
| `Group` | Run as this group | `www-data` |
| `WorkingDirectory` | Set working directory | `/var/www/app` |
| `Environment` | Set environment variables | `DATABASE_URL=postgres://...` |
| `EnvironmentFile` | Load env vars from file | `/etc/app/env` |

---

### **Service Types Explained**

| Type | Behavior | Use Case |
|------|----------|----------|
| `simple` | ExecStart is the main process | Most modern apps (Python, Node.js) |
| `forking` | Process forks, parent exits | Traditional daemons (nginx, Apache) |
| `oneshot` | Runs once then exits | Scripts, initialization tasks |
| `notify` | Sends notification when ready | Apps using sd_notify() |

**simple (most common):**
```ini
[Service]
Type=simple
ExecStart=/usr/bin/python app.py
# systemd tracks the python process directly
```

**forking (traditional daemons):**
```ini
[Service]
Type=forking
PIDFile=/run/nginx.pid
ExecStart=/usr/sbin/nginx
# nginx forks: parent exits, child runs in background
# systemd reads PID from PIDFile to track the child
```

---

### **[Install] Section - Boot Configuration**

| Directive | Purpose | Common Values |
|-----------|---------|---------------|
| `WantedBy` | Which target "wants" this service | `multi-user.target` |
| `RequiredBy` | Which target "requires" this service | (rarely used) |
| `Alias` | Alternative names | `Alias=webapp.service` |

```ini
[Install]
WantedBy=multi-user.target
# When you run "systemctl enable", creates symlink in:
# /etc/systemd/system/multi-user.target.wants/
```

---

## 🛠️ **CREATING YOUR OWN SERVICE**

### **Scenario: Deploy Flask App as a Service**

You have a Flask application that needs to:
- Run with Gunicorn
- Start on boot
- Restart if it crashes
- Run as non-root user

---

### **Step 1: Create the Unit File**

```bash
$ sudo nano /etc/systemd/system/flask-app.service
```

```ini
[Unit]
Description=Flask Application (Gunicorn)
Documentation=https://internal-docs.example.com/flask-app
After=network.target postgresql.service
Requires=postgresql.service

[Service]
Type=simple
User=www-data
Group=www-data
WorkingDirectory=/var/www/flask-app

# Environment variables
Environment="FLASK_ENV=production"
Environment="DATABASE_URL=postgresql://localhost/mydb"
# Or load from file:
# EnvironmentFile=/etc/flask-app/env

# Start command
ExecStart=/var/www/flask-app/venv/bin/gunicorn \
    --workers 4 \
    --bind 127.0.0.1:5000 \
    --access-logfile /var/log/flask-app/access.log \
    --error-logfile /var/log/flask-app/error.log \
    app:app

# Restart policy
Restart=always
RestartSec=5

# Security hardening
PrivateTmp=true
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

---

### **Step 2: Prepare the Environment**

```bash
# Create log directory
$ sudo mkdir -p /var/log/flask-app
$ sudo chown www-data:www-data /var/log/flask-app

# Ensure app directory permissions
$ sudo chown -R www-data:www-data /var/www/flask-app
```

---

### **Step 3: Load and Start**

```bash
# Tell systemd about the new unit file
$ sudo systemctl daemon-reload

# Start the service
$ sudo systemctl start flask-app

# Check status
$ systemctl status flask-app
● flask-app.service - Flask Application (Gunicorn)
     Loaded: loaded (/etc/systemd/system/flask-app.service; disabled)
     Active: active (running) since Thu 2026-01-23 10:00:00 UTC
   Main PID: 12345 (gunicorn)
     ...

# Test it works
$ curl http://127.0.0.1:5000/health
{"status": "ok"}

# Enable for boot
$ sudo systemctl enable flask-app
```

---

### **Step 4: Verify Auto-Restart**

```bash
# Find the main PID
$ systemctl status flask-app | grep "Main PID"
   Main PID: 12345 (gunicorn)

# Kill it (simulate crash)
$ sudo kill -9 12345

# Wait 5 seconds (RestartSec=5)...

# Check - it should be running with NEW PID
$ systemctl status flask-app
● flask-app.service - Flask Application (Gunicorn)
     Active: active (running) since Thu 2026-01-23 10:00:05 UTC
   Main PID: 12350 (gunicorn)  ← New PID!
```

**systemd automatically restarted the crashed service!**

---

## 📊 **VIEWING LOGS WITH journalctl**

### **Why journalctl?**

systemd captures ALL output from services into a binary journal. Much better than scattered log files!

```
Traditional:
/var/log/nginx/error.log
/var/log/postgresql/postgresql.log
/var/log/flask-app/error.log
└─ Different formats, different locations

systemd journal:
journalctl -u nginx
journalctl -u postgresql
journalctl -u flask-app
└─ Same command, same format, timestamps aligned
```

---

### **Basic journalctl Usage**

```bash
# All logs for a service
$ journalctl -u nginx

# Follow logs in real-time (like tail -f)
$ journalctl -u flask-app -f

# Last 50 lines
$ journalctl -u nginx -n 50

# Logs since last boot
$ journalctl -u nginx -b

# Logs from specific time
$ journalctl -u nginx --since "2026-01-23 10:00:00"
$ journalctl -u nginx --since "1 hour ago"
$ journalctl -u nginx --since today

# Logs between times
$ journalctl -u nginx --since "2026-01-23 10:00" --until "2026-01-23 11:00"
```

---

### **Filtering by Priority**

```bash
# Only errors and above
$ journalctl -u flask-app -p err

# Priority levels:
# 0 - emerg
# 1 - alert
# 2 - crit
# 3 - err
# 4 - warning
# 5 - notice
# 6 - info
# 7 - debug

# Errors and warnings
$ journalctl -u flask-app -p warning
```

---

### **Real-World Debugging: "Why Did It Crash?"**

```bash
# Service is failed
$ systemctl status flask-app
● flask-app.service - Flask Application
     Active: failed (Result: exit-code) since...

# Check the logs around the crash
$ journalctl -u flask-app -n 100

# Output shows:
Jan 23 10:45:00 server gunicorn[12345]: [ERROR] Connection refused: postgresql
Jan 23 10:45:00 server gunicorn[12345]: Unable to connect to database
Jan 23 10:45:00 server systemd[1]: flask-app.service: Main process exited, code=exited, status=1/FAILURE
Jan 23 10:45:05 server systemd[1]: flask-app.service: Scheduled restart job

# Root cause: Database connection failed!
$ systemctl status postgresql
○ postgresql.service - inactive (dead)
# PostgreSQL is down - fix that first
```

---

### **JSON Output (for Scripts/Automation)**

```bash
# Output as JSON
$ journalctl -u nginx -o json-pretty -n 1

{
    "__CURSOR" : "s=...",
    "MESSAGE" : "Starting nginx...",
    "_SYSTEMD_UNIT" : "nginx.service",
    "_PID" : "1234",
    "__REALTIME_TIMESTAMP" : "1737626400000000"
}

# Parse with jq
$ journalctl -u nginx -o json | jq -r '.MESSAGE'
```

---

## 🔧 **EDITING AND OVERRIDING SERVICES**

### **The Problem**

```bash
# You want to change nginx's memory limit
# But /lib/systemd/system/nginx.service is managed by apt!
# If you edit it, package updates will overwrite your changes
```

---

### **Solution: Drop-in Overrides**

```bash
# Create override directory
$ sudo systemctl edit nginx
# Opens editor with blank file

# Add your overrides:
[Service]
MemoryLimit=512M
```

This creates: `/etc/systemd/system/nginx.service.d/override.conf`

```bash
# Apply changes
$ sudo systemctl daemon-reload
$ sudo systemctl restart nginx
```

---

### **View Effective Configuration**

```bash
# See final merged configuration
$ systemctl cat nginx

# Shows:
# /lib/systemd/system/nginx.service (original)
# /etc/systemd/system/nginx.service.d/override.conf (your overrides)
```

---

### **Common Overrides**

**Increase timeout:**
```ini
[Service]
TimeoutStartSec=300
TimeoutStopSec=300
```

**Change restart behavior:**
```ini
[Service]
Restart=always
RestartSec=10
```

**Add environment variable:**
```ini
[Service]
Environment="DEBUG=true"
```

**Change resource limits:**
```ini
[Service]
MemoryLimit=1G
CPUQuota=50%
```

---

## 🛠️ **PRACTICAL PRODUCTION SCENARIOS**

### **Scenario 1: "Service Won't Start!"**

```bash
$ sudo systemctl start flask-app
Job for flask-app.service failed because the control process exited with error code.

# Step 1: Check status
$ systemctl status flask-app
● flask-app.service - Flask Application
     Loaded: loaded (/etc/systemd/system/flask-app.service; enabled)
     Active: failed (Result: exit-code) since...
    Process: 12345 ExecStart=/var/www/flask-app/venv/bin/gunicorn... (code=exited, status=1)

# Step 2: Check logs
$ journalctl -u flask-app -n 50
Jan 23 10:00:00 server gunicorn[12345]: ModuleNotFoundError: No module named 'flask'

# Root cause: Virtual environment missing flask!
# Fix:
$ sudo -u www-data /var/www/flask-app/venv/bin/pip install flask gunicorn
$ sudo systemctl start flask-app
```

---

### **Scenario 2: "Service Keeps Restarting!"**

```bash
# Notice rapid restarts
$ systemctl status flask-app
     Active: activating (auto-restart)...

# Check restart count
$ systemctl show flask-app | grep NRestarts
NRestarts=15

# Check logs for pattern
$ journalctl -u flask-app --since "5 minutes ago"
# See same error repeating every 5 seconds

# Temporarily stop restart loop to debug
$ sudo systemctl stop flask-app

# Debug manually
$ sudo -u www-data /var/www/flask-app/venv/bin/gunicorn app:app
# See actual error, fix it

# Then restart
$ sudo systemctl start flask-app
```

---

### **Scenario 3: "Zero-Downtime Deploy"**

```bash
# Update code
$ cd /var/www/flask-app
$ git pull origin main

# Reload (if app supports it) OR restart gracefully
$ sudo systemctl reload flask-app
# OR
$ sudo systemctl restart flask-app

# Verify
$ systemctl status flask-app
$ curl http://localhost:5000/health
```

---

### **Scenario 4: "What's Using Port 5000?"**

```bash
# Find what's on port 5000
$ sudo ss -tlnp | grep 5000
LISTEN  0  128  127.0.0.1:5000  *  users:(("gunicorn",pid=12345,fd=5))

# Find service by PID
$ systemctl status 12345
● flask-app.service - Flask Application
```

---

## 🧪 **MINI-TASK: Create and Manage a Service**

### **Objective:** 
Create a systemd service for a simple application, manage it, and troubleshoot issues.

---

### **Part 1: Create an Application**

Create a simple Python script to run as a service:

```bash
# Create directory
mkdir -p ~/webapp
cd ~/webapp

# Create the application
cat > app.py << 'EOF'
#!/usr/bin/env python3
import time
import signal
import sys
from datetime import datetime

def signal_handler(signum, frame):
    print(f"Received signal {signum}, shutting down gracefully...")
    sys.exit(0)

signal.signal(signal.SIGTERM, signal_handler)
signal.signal(signal.SIGHUP, lambda s, f: print("Received SIGHUP - would reload config here"))

print(f"Application starting at {datetime.now()}")
print(f"PID: {os.getpid()}")

counter = 0
while True:
    counter += 1
    print(f"[{datetime.now()}] Heartbeat #{counter}")
    time.sleep(10)
EOF

# Add missing import
sed -i '1a import os' ~/webapp/app.py

# Make executable
chmod +x ~/webapp/app.py

# Test it runs
python3 ~/webapp/app.py
# Ctrl+C to stop
```

---

### **Part 2: Create the Service**

Create `/etc/systemd/system/myapp.service` with:
- Proper description
- Runs as your user
- WorkingDirectory set correctly
- Restarts on failure
- Environment variable `APP_ENV=production`

---

### **Part 3: Tasks**

1. **Basic Management:**
   - Start the service
   - Check its status (note the PID)
   - View logs with journalctl
   - Stop the service

2. **Boot Persistence:**
   - Enable the service for boot
   - Verify with `is-enabled`
   - Disable it

3. **Signal Handling:**
   - Start the service
   - Send SIGHUP using `systemctl kill -s HUP myapp`
   - Check logs to see "would reload config"
   - Stop gracefully, verify "shutting down gracefully" in logs

4. **Auto-Restart Test:**
   - Start the service
   - Kill it with `kill -9 <PID>`
   - Verify systemd restarted it (new PID)

5. **Override Configuration:**
   - Use `systemctl edit myapp` to add:
     - `RestartSec=2` (faster restart)
     - `Environment="DEBUG=true"`
   - Apply and verify changes

---

### **Part 4: Create Report**

Create `/home/sam/DevopsRoadmap/Phase-1-Linux/systemd-report.txt` containing:

1. Your complete unit file content
2. Output of `systemctl status myapp` (while running)
3. Output of `journalctl -u myapp -n 20`
4. Proof of auto-restart (old PID vs new PID)
5. Content of your override file

---

### **Security Questions:**

1. Why should services run as non-root users?
2. What's the difference between `Wants=` and `Requires=`?
3. When would you use `Type=forking` vs `Type=simple`?
4. How do drop-in overrides help with package updates?

---

**Expected Time:** 60-90 minutes

**Deliverable:** Show me your `systemd-report.txt` and explain what you learned.

---

## 📌 **Quick Reference Card**

```bash
# Status & Info
systemctl status nginx           # Detailed status
systemctl is-active nginx        # Just "active" or "inactive"
systemctl is-enabled nginx       # Just "enabled" or "disabled"
systemctl list-units --type=service  # All loaded services
systemctl list-unit-files        # All available services

# Control
sudo systemctl start nginx       # Start now
sudo systemctl stop nginx        # Stop now
sudo systemctl restart nginx     # Stop then start
sudo systemctl reload nginx      # Reload config (no restart)
sudo systemctl reload-or-restart nginx  # Reload if possible

# Boot Persistence
sudo systemctl enable nginx      # Start on boot
sudo systemctl disable nginx     # Don't start on boot
sudo systemctl enable --now nginx  # Enable AND start

# Logs
journalctl -u nginx              # All logs for service
journalctl -u nginx -f           # Follow (real-time)
journalctl -u nginx -n 50        # Last 50 lines
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -p err       # Only errors

# Unit Files
systemctl cat nginx              # View unit file
sudo systemctl edit nginx        # Create override
sudo systemctl daemon-reload     # Reload unit files after changes

# Debugging
systemctl show nginx             # All properties
systemctl list-dependencies nginx  # Show dependencies
systemctl --failed               # List failed services
```

---

**Complete this mini-task, then we'll move to Module 5: Users, Groups & sudo.**
