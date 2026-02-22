# Module 8: Monitoring & Logs

## 🎯 Why This Matters in Production

**Scenario**: You just deployed a new version of your app. 10 minutes later, the on-call alarm goes off. Users are reporting errors. You have no GUI, no fancy dashboard — just SSH access.

You need to answer in under 5 minutes:
- **Is it the app, the OS, or the hardware?**
- **When did it start breaking?**
- **Is the system running out of memory/CPU/disk?**
- **What exactly went wrong?**

**Monitoring & logs = your eyes when the system is on fire.**

---

## 🧠 **THE OBSERVABILITY STACK**

```
┌────────────────────────────────────────────────────────────────────────┐
│                    LINUX OBSERVABILITY LAYERS                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   LAYER 4: APPLICATION LOGS                                            │
│   /var/log/app.log, /var/log/nginx/access.log, journald               │
│   "What did MY software do?"                                           │
│                                                                         │
│   LAYER 3: SERVICE LOGS                                                │
│   journalctl, /var/log/syslog, /var/log/auth.log                       │
│   "What did system services do?"                                       │
│                                                                         │
│   LAYER 2: KERNEL & HARDWARE LOGS                                      │
│   dmesg, /var/log/kern.log                                             │
│   "What did the kernel/hardware do?"                                   │
│                                                                         │
│   LAYER 1: PERFORMANCE METRICS                                         │
│   top, htop, vmstat, iostat, free, df, uptime                         │
│   "How are the resources doing RIGHT NOW?"                             │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 📊 **REAL-TIME PERFORMANCE MONITORING**

### **uptime — System Load at a Glance**

```bash
uptime
# Output: 14:32:11 up 3 days,  2:14,  2 users,  load average: 0.52, 0.71, 0.85
#                                                               1min  5min  15min
```

**Understanding Load Average:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  Load average = average number of processes WAITING for CPU         │
│                                                                     │
│  Rule of thumb: compare load to number of CPU cores                │
│                                                                     │
│  4-core system:                                                     │
│    load = 0.5  → 12.5% utilization  ✅  healthy                   │
│    load = 4.0  → 100% utilization   ⚠️  at capacity               │
│    load = 8.0  → 200% utilization   🔥  overloaded                │
│                                                                     │
│  Check core count: nproc   OR   grep -c processor /proc/cpuinfo    │
└─────────────────────────────────────────────────────────────────────┘
```

```bash
# Check CPU core count
nproc
grep -c processor /proc/cpuinfo

# Load average interpretation
uptime | awk '{print $NF}' | cut -d',' -f1   # 15-min load
```

---

### **top — Live Process Monitor**

```bash
top
```

**Anatomy of top output:**

```
top - 14:32:11 up 3 days, load average: 0.52, 0.71, 0.85
Tasks: 212 total,   1 running, 211 sleeping
%Cpu(s):  5.2 us,  1.3 sy,  0.0 ni, 92.8 id,  0.4 wa
MiB Mem :   5944.0 total,    812.3 free,   3201.4 used,   1930.3 buff/cache
MiB Swap:   2048.0 total,   2010.7 free,     37.3 used.   2413.7 avail Mem

  PID USER  PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 1234 nginx 20   0  121MB   12MB   8MB  S   2.0   0.2   0:12.44 nginx
```

**CPU line decoded:**
| Field | Meaning |
|-------|---------|
| `us` | User space (your apps) |
| `sy` | Kernel/system |
| `ni` | Niced processes |
| `id` | Idle (what's left = good) |
| `wa` | **I/O wait — if high, disk is your bottleneck** |
| `st` | Stolen (VM host taking CPU from you) |

**Memory line decoded:**
| Field | Meaning |
|-------|---------|
| `total` | Physical RAM |
| `free` | Truly unused |
| `used` | In use |
| `buff/cache` | OS using for cache (can be freed if needed) |
| `avail` | **What's actually available for new processes** |

> ⚠️ Don't panic at high `used` memory! Check `avail` instead. Linux deliberately uses free RAM as disk cache — this is a feature, not a problem.

**Useful top keyboard shortcuts:**
| Key | Action |
|-----|--------|
| `P` | Sort by CPU |
| `M` | Sort by memory |
| `k` | Kill a process (type PID) |
| `r` | Renice a process |
| `1` | Show per-core CPU usage |
| `q` | Quit |
| `h` | Help |

---

### **htop — Better top (Interactive)**

```bash
htop
```

Advantages over `top`:
- Mouse support
- Color-coded bars
- Easier process tree (`F5`)
- Easier kill (`F9`)
- Filter processes (`F4`)

---

### **free — Memory Usage**

```bash
free -h        # Human readable
free -m        # In megabytes
free -s 2      # Refresh every 2 seconds
```

**Output:**
```
              total        used        free      shared  buff/cache   available
Mem:           5.8G        3.1G        793M        201M        1.9G        2.4G
Swap:          2.0G         36M        2.0G
```

**What to watch:**
- `available` going below 200MB → danger zone
- `Swap used` growing → system is out of RAM and using disk (very slow)
- `Swap used` = 0 → healthy, all in RAM

```bash
# Watch memory every 2 seconds
watch -n 2 free -h

# Quick check: is swap being used?
free -h | awk '/Swap/ {print "Swap used:", $3}'
```

---

### **df — Disk Space**

```bash
df -h           # Human readable, all filesystems
df -h /         # Just root filesystem
df -i           # Inode usage (a hidden disk-full cause!)
```

**Output:**
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        98G   43G   50G  47% /
tmpfs           2.9G  1.2M  2.9G   1% /dev/shm
```

> 🚨 **Hidden trap: Inodes!** A disk can be 0% full by size but 100% full by inodes (if millions of tiny files). Always check both `df -h` AND `df -i`.

```bash
# Check if disk is full
df -h | awk 'NR>1 && $5+0 > 80 {print "WARNING:", $5, $6}'

# Check inode usage
df -i | awk 'NR>1 && $5+0 > 80 {print "INODE WARNING:", $5, $6}'
```

---

### **du — Disk Usage by Directory**

```bash
du -sh /var/log          # Size of a directory
du -sh /*                # Size of all top-level dirs
du -ah /var/log          # All files with sizes
du -ah /var/log | sort -rh | head -20   # Top 20 largest
```

**Finding what's eating your disk:**
```bash
# Step 1: Find the biggest directory at root
du -sh /* 2>/dev/null | sort -rh | head -10

# Step 2: Drill down
du -sh /var/* 2>/dev/null | sort -rh | head -10

# Step 3: Find the culprit files
du -ah /var/log 2>/dev/null | sort -rh | head -20
```

---

### **vmstat — Virtual Memory Statistics**

```bash
vmstat             # Snapshot
vmstat 2           # Refresh every 2 seconds
vmstat 2 5         # 5 samples, every 2 seconds
```

**Output:**
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 810000  12000 1900000   0    0    10    20  200  400  5  2 92  1  0
```

**Key columns:**
| Column | Meaning | Watch if... |
|--------|---------|-------------|
| `r` | Processes waiting for CPU | > number of cores |
| `b` | Processes in uninterruptible sleep | > 0 for extended time |
| `si/so` | Swap in/out (KB/s) | > 0 means RAM pressure |
| `bi/bo` | Block I/O (blocks/s) | Very high = disk bottleneck |
| `wa` | CPU waiting for I/O | > 10% = I/O bottleneck |
| `us` | User CPU | Consistently near 100% = CPU bound |

---

### **iostat — I/O Statistics**

```bash
# May need: sudo apt install sysstat
iostat            # Snapshot
iostat -x         # Extended (more detail)
iostat -x 2       # Every 2 seconds
iostat -xz 2 5    # Extended, skip idle devices
```

**Output:**
```
Device    r/s    w/s   rMB/s  wMB/s  await  %util
sda      10.5   45.2    0.5    2.3   12.5   35.0
```

**Key columns:**
| Column | Meaning | Concern if... |
|--------|---------|--------------|
| `r/s, w/s` | Read/write ops per second | Depends on device |
| `rMB/s, wMB/s` | Throughput | Near device limit |
| `await` | Average wait time (ms) | > 20ms for SSD, > 100ms for HDD |
| `%util` | Disk utilization | > 80% = bottleneck |

---

## 📋 **SYSTEMD LOGS — journalctl**

`journalctl` is the gateway to ALL systemd-managed logs. It's the most important log tool in modern Ubuntu/Debian systems.

### **Basic journalctl Usage**

```bash
# View all logs (oldest first)
journalctl

# View all logs (newest first) — much more useful!
journalctl -r

# Follow live (like tail -f)
journalctl -f

# Last N lines
journalctl -n 50

# Last N lines, follow
journalctl -n 50 -f
```

### **Filter by Service**

```bash
# Logs for a specific service
journalctl -u nginx
journalctl -u ssh
journalctl -u postgresql

# Follow a service's logs
journalctl -u nginx -f

# Last 50 lines from a service
journalctl -u nginx -n 50

# Multiple services
journalctl -u nginx -u postgresql
```

### **Filter by Time**

```bash
# Since a specific time
journalctl --since "2026-02-22 10:00:00"

# Until a specific time
journalctl --until "2026-02-22 11:00:00"

# Time range
journalctl --since "2026-02-22 10:00:00" --until "2026-02-22 11:00:00"

# Relative time shortcuts
journalctl --since "1 hour ago"
journalctl --since "yesterday"
journalctl --since "today"

# Last boot
journalctl -b

# Previous boot (useful after crash!)
journalctl -b -1
```

### **Filter by Priority**

```bash
# Priority levels (0=emergency → 7=debug)
journalctl -p err           # Error and above
journalctl -p warning       # Warning and above
journalctl -p info          # Info and above

# Priority numbers work too
journalctl -p 3             # err (0=emerg, 1=alert, 2=crit, 3=err)
```

```
┌─────────────────────────────────────────────────────────────────────┐
│  SYSLOG PRIORITY LEVELS                                             │
├──────┬───────────┬───────────────────────────────────────────────  │
│  0   │ emerg     │ System unusable                                 │
│  1   │ alert     │ Immediate action required                       │
│  2   │ crit      │ Critical condition                              │
│  3   │ err       │ Error condition                                 │
│  4   │ warning   │ Warning condition                               │
│  5   │ notice    │ Normal but significant                          │
│  6   │ info      │ Informational                                   │
│  7   │ debug     │ Debug-level messages                            │
└──────┴───────────┴───────────────────────────────────────────────  │
└─────────────────────────────────────────────────────────────────────┘
```

### **Useful journalctl Combos**

```bash
# Errors in the last hour
journalctl -p err --since "1 hour ago"

# nginx errors since yesterday
journalctl -u nginx -p err --since "yesterday"

# All kernel messages from last boot
journalctl -b -k

# Disk-related messages
journalctl -k | grep -i "disk\|sda\|nvme"

# OOM (Out of Memory) killer events — very important!
journalctl -k | grep -i "oom\|killed process"

# Show logs with full output (no truncation)
journalctl -u nginx --no-pager

# Export logs for analysis
journalctl -u nginx --since "today" --no-pager > /tmp/nginx-today.log

# Show disk usage of journal
journalctl --disk-usage

# Vacuum old logs (keep last 2 weeks)
sudo journalctl --vacuum-time=2weeks
```

---

## 📁 **TRADITIONAL LOG FILES**

Not everything uses systemd. Many applications write directly to `/var/log/`.

### **Key Log Files**

```
┌──────────────────────────────────────────────────────────────────────┐
│  /var/log/                                                           │
├─────────────────────────┬────────────────────────────────────────── │
│  syslog                 │ General system messages (Ubuntu)          │
│  auth.log               │ Authentication: SSH logins, sudo          │
│  kern.log               │ Kernel messages                           │
│  dpkg.log               │ Package installs/removals                 │
│  apt/history.log        │ apt command history                       │
│  nginx/access.log       │ HTTP requests                             │
│  nginx/error.log        │ nginx errors                              │
│  postgresql/            │ Database logs                             │
│  mail.log               │ Mail server logs                          │
│  ufw.log                │ Firewall logs                             │
└─────────────────────────┴────────────────────────────────────────── │
└──────────────────────────────────────────────────────────────────────┘
```

### **Reading Log Files**

```bash
# View entire log
cat /var/log/syslog

# Live tail
tail -f /var/log/syslog

# Last 100 lines with follow
tail -100f /var/log/auth.log

# Search for errors
grep -i "error\|fail\|crit" /var/log/syslog | tail -50

# View auth log for SSH logins
grep "Accepted password\|Accepted publickey" /var/log/auth.log

# View failed logins
grep "Failed password\|authentication failure" /var/log/auth.log

# Multiple log files live
tail -f /var/log/syslog /var/log/auth.log
```

---

## 🔧 **dmesg — Kernel Ring Buffer**

`dmesg` shows kernel messages — hardware events, driver issues, kernel errors.

```bash
# View all kernel messages
dmesg

# Human-readable timestamps
dmesg -T

# Follow live kernel messages
dmesg -w
dmesg -Tw          # With timestamps

# Filter by level
dmesg -l err       # Errors only
dmesg -l warn      # Warnings only
dmesg -l err,warn  # Both

# Recent messages (last 20)
dmesg -T | tail -20

# Search for specific hardware
dmesg | grep -i "usb\|disk\|sda\|nvme"

# Memory errors
dmesg | grep -i "memory\|mce\|edac"

# OOM killer
dmesg | grep -i "oom\|killed"

# Network interface messages
dmesg | grep -i "eth0\|ens\|wlan"
```

**Common dmesg scenarios:**

```bash
# Check if disk has errors
dmesg -T | grep -i "I/O error\|hard resetting"

# Check if NIC dropped packets
dmesg -T | grep -i "dropped\|fifo"

# Check USB device events
dmesg -T | grep -i "usb\|USB"

# Check filesystem errors
dmesg -T | grep -i "EXT4\|XFS\|btrfs\|filesystem"
```

---

## 🔄 **LOG ROTATION — logrotate**

Log files grow forever if not managed. `logrotate` automatically rotates, compresses, and deletes old logs.

### **How it works:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  LOG ROTATION LIFECYCLE                                              │
│                                                                      │
│  app.log         ← current (being written to)                       │
│  app.log.1       ← yesterday's                                      │
│  app.log.2.gz    ← 2 days ago (compressed)                          │
│  app.log.3.gz    ← 3 days ago                                       │
│  app.log.4.gz    ← ...                                              │
│       ...                                                            │
│  [deleted]       ← older than retention period                      │
└──────────────────────────────────────────────────────────────────────┘
```

### **logrotate Configuration**

Config files live in:
- `/etc/logrotate.conf` — global defaults
- `/etc/logrotate.d/` — per-application configs

```bash
# View global config
cat /etc/logrotate.conf

# View application configs
ls /etc/logrotate.d/
cat /etc/logrotate.d/nginx
```

**Example config:**
```
/var/log/nginx/*.log {
    daily           # Rotate daily
    missingok       # Don't error if log missing
    rotate 14       # Keep 14 rotated logs
    compress        # Compress old logs
    delaycompress   # Compress after 1 rotation (so current-1 is readable)
    notifempty      # Don't rotate empty files
    create 0640 www-data adm   # Create new log with permissions
    sharedscripts
    postrotate      # Run after rotation
        nginx -s reopen
    endscript
}
```

```bash
# Test logrotate config (dry run)
sudo logrotate -d /etc/logrotate.d/nginx

# Force rotation now (for testing)
sudo logrotate -f /etc/logrotate.d/nginx

# Check logrotate status
cat /var/lib/logrotate/status
```

---

## 🚨 **PRODUCTION MONITORING WORKFLOWS**

### **Workflow 1: System is Slow — Find the Bottleneck**

```bash
# Step 1: Check load average
uptime
# If load >> nproc, you have a bottleneck. Go to Step 2.

# Step 2: Is it CPU or I/O?
vmstat 1 5
# If 'wa' (wait) > 10% → I/O bottleneck → go to iostat
# If 'us' or 'sy' high → CPU bottleneck → go to top

# Step 3a: CPU bottleneck — find the process
top -o %CPU
# Press P for CPU sort, identify the PID

# Step 3b: I/O bottleneck — find the disk
iostat -x 2
# High %util on a device → that disk is the problem

# Step 4: Check if it's memory pressure causing I/O
free -h
vmstat 1 5
# If swap si/so > 0 → system is swapping → add RAM or kill a process
```

### **Workflow 2: Application Crashed — What Happened?**

```bash
# Step 1: Check service status
systemctl status myapp

# Step 2: Check recent logs
journalctl -u myapp -n 100 --no-pager

# Step 3: Check logs around crash time
journalctl -u myapp --since "30 minutes ago"

# Step 4: Check if OOM killer struck
journalctl -k | grep -i "oom\|killed"
dmesg -T | grep -i "killed process"

# Step 5: Check system errors at the same time
journalctl -p err --since "30 minutes ago"

# Step 6: Check previous boot if server rebooted
journalctl -b -1 -p err
```

### **Workflow 3: Disk Full — Emergency Response**

```bash
# Step 1: Confirm and find where
df -h
# Which filesystem is full?

# Step 2: Find what's consuming it
du -sh /* 2>/dev/null | sort -rh | head
# Drill down into the culprit directory

# Step 3: Check for large log files
find /var/log -size +100M -type f
ls -lhS /var/log/*.log 2>/dev/null | head

# Step 4: Check for deleted-but-open files (common!)
lsof +L1 2>/dev/null | grep -v "^COMMAND"
# Files deleted but still held open by processes don't free space until process dies

# Step 5: Quick wins
sudo apt clean                    # Clear package cache
sudo journalctl --vacuum-size=500M  # Trim journal logs
sudo find /tmp -type f -atime +7 -delete  # Clear old tmp files
```

### **Workflow 4: Security Check — Who Did What?**

```bash
# Who's logged in right now?
who
w

# Recent logins
last | head -20

# Failed login attempts
grep "Failed password" /var/log/auth.log | tail -20

# Brute force attack detection
grep "Failed password" /var/log/auth.log | \
  grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | \
  sort | uniq -c | sort -rn | head

# Successful logins from unusual IPs
grep "Accepted" /var/log/auth.log | \
  awk '{print $11}' | sort | uniq -c | sort -rn

# What commands did a user run with sudo?
grep "sudo" /var/log/auth.log | grep "sam" | tail -20
```

---

## 🧮 **WATCH — Run Commands Periodically**

```bash
# Run command every 2 seconds (default)
watch df -h

# Custom interval
watch -n 5 free -h

# Highlight changes
watch -d free -h

# One expression
watch -n 1 'ps aux | grep nginx | grep -v grep'

# Monitor log file size
watch -n 5 'ls -lh /var/log/app.log'
```

---

## ⚠️ **COMMON MISTAKES**

### **1. Confusing `free` and `available` Memory**

```bash
# WRONG interpretation
free -h | grep Mem
# Mem:  5.8G  4.9G  200M ...
# "Only 200MB free! We're out of memory!" ← WRONG

# RIGHT interpretation — look at 'available'
# Mem:  5.8G  4.9G  200M ... 2.4G
#                             ^
#                          2.4G actually available (kernel can release cache)
```

### **2. Ignoring I/O Wait**

```bash
# Many people see 95% idle CPU and think "no CPU problem"
# But wa=25% means CPU spends 25% doing nothing, WAITING FOR DISK
# This is still a bottleneck!

top  # Check the wa column in the %Cpu line
```

### **3. Not Checking Inodes**

```bash
# Disk shows 40% full, but you can't create new files?
df -i    # Check inode usage — might be 100%!
```

### **4. `tail -f` vs `tail -F`**

```bash
# -f: follows the file by file descriptor
# If the file is rotated (deleted and recreated), -f stops getting updates

# -F: follows by filename, handles rotation automatically
tail -F /var/log/nginx/access.log   # Always use -F in production
```

### **5. Forgetting `--no-pager` in Scripts**

```bash
# journalctl opens a pager (like less) by default — will hang in scripts!
journalctl -u nginx | grep error   # HANGS

# Fix
journalctl -u nginx --no-pager | grep error
# OR
journalctl -u nginx | grep error --no-pager
```

---

## 📝 **QUICK REFERENCE**

```bash
# ============== PERFORMANCE ==============
uptime                             # Load average
nproc                              # CPU core count
top / htop                         # Live process monitor
free -h                            # Memory usage
df -h                              # Disk space
df -i                              # Inode usage
du -sh /path                       # Directory size
vmstat 2 5                         # VM + I/O stats
iostat -x 2                        # Disk I/O detail

# ============== JOURNALCTL ==============
journalctl -f                      # Follow all logs
journalctl -u nginx -f             # Follow service logs
journalctl -u nginx -n 100         # Last 100 lines
journalctl -p err                  # Errors only
journalctl --since "1 hour ago"    # Since time
journalctl -b                      # Current boot
journalctl -b -1                   # Previous boot
journalctl --disk-usage            # Journal size
journalctl --vacuum-time=2weeks    # Clean old journals

# ============== LOG FILES ==============
tail -F /var/log/syslog            # Follow syslog
grep "error" /var/log/syslog       # Search logs
grep "Accepted" /var/log/auth.log  # SSH logins
grep "Failed" /var/log/auth.log    # Failed logins

# ============== DMESG ==============
dmesg -T                           # Kernel msgs with timestamps
dmesg -Tw                          # Follow kernel msgs
dmesg -l err,warn                  # Errors and warnings
dmesg | grep -i "oom\|killed"      # OOM killer events

# ============== WATCH ==============
watch -n 2 free -h                 # Monitor memory
watch -n 5 df -h                   # Monitor disk
watch -d -n 2 'command'            # Highlight changes
```

---

## 🎯 **MINI-TASK: System Health Report**

### **Part 1: Performance Snapshot**

Run these and save output:

1. **Load check**: `uptime` — what is the load average? How does it compare to your core count?

2. **CPU breakdown**: Run `top -bn1 | head -20` (non-interactive, 1 batch) — what % is idle?

3. **Memory status**: `free -h` — how much is available vs total?

4. **Disk space**: `df -h` — which filesystems exist and what's their usage?

5. **Disk inodes**: `df -i` — are any at risk?

6. **Biggest directories**: `du -sh /* 2>/dev/null | sort -rh | head -10`

7. **I/O stats**: `vmstat 2 3` — is there any swap I/O (si/so)?

### **Part 2: Log Investigation**

8. **Boot messages**: `journalctl -b -p warning --no-pager | tail -30`
   — Any warnings or errors since last boot?

9. **Recent system errors**: `journalctl -p err --since "today" --no-pager | tail -20`

10. **Auth log check**: `grep -E "Accepted|Failed" /var/log/auth.log | tail -20`
    — Any login attempts you recognise? Any failed ones?

11. **Kernel messages**: `dmesg -T | tail -30`
    — Any hardware or kernel warnings?

12. **Service health check**: Check status of 3 services: `ssh`, `systemd-logind`, and one more of your choice
    ```bash
    systemctl status ssh --no-pager
    systemctl status systemd-logind --no-pager
    ```

### **Part 3: Disk Housekeeping**

13. **Journal size**: `journalctl --disk-usage`

14. **Package cache**: `du -sh /var/cache/apt/archives/`

15. **Log file sizes**: `ls -lhS /var/log/*.log 2>/dev/null | head -10`

### **Security Questions:**

1. What does a load average of `8.00` mean on a 4-core system?
2. What's the difference between `free` memory and `available` memory?
3. Why should you use `tail -F` instead of `tail -f` in production?
4. What does it mean when `vmstat` shows `si/so > 0`?
5. What is the OOM killer and when does it activate?

---

Create `/home/sam/DevopsRoadmap/Phase-1-Linux/monitoring-report.txt` with all commands AND their outputs. Show your thinking where asked!
