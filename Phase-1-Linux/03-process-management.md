# Module 3: Process Management

## 🎯 Why This Matters in Production

**Scenario**: Your web application is consuming 100% CPU. Users are complaining the site is slow. Your manager asks: "What's going on?"

You need to instantly know:
- **What processes are running?**
- **Which one is consuming resources?**
- **How do I kill the problematic process?**
- **How do I restart it properly?**

**In production, understanding processes = understanding your running system.**

---

## 🧠 **WHAT IS A PROCESS? (THE FOUNDATION)**

### **Definition**

A **process** is a **running instance of a program**.

```
Program (on disk)  →  Execute  →  Process (in memory)
     │                              │
     ├─ Stored file                 ├─ Running code
     ├─ Executable code             ├─ Allocated memory
     └─ Static                      ├─ CPU time
                                    ├─ File handles
                                    └─ Dynamic
```

**Simple analogy:**
- **Program** = Recipe (instructions on paper)
- **Process** = Actually cooking the dish (recipe in action)

---

### **Key Concepts**

Every process has:

1. **PID (Process ID)** - Unique number identifying the process
2. **PPID (Parent Process ID)** - The process that started this one
3. **Owner** - User who started it
4. **State** - Running, sleeping, stopped, zombie
5. **Priority** - How important it is (CPU scheduling)
6. **Memory** - RAM it's using
7. **CPU usage** - Percentage of CPU time

---

### 🎯 **Running Example: Web Application Stack**

Throughout this module, we'll use this **real-world scenario**:

```
You're running a web application stack:

┌─────────────────────────────────────────┐
│         NGINX Web Server                │
│         (receives HTTP requests)        │
│         PID: 1234                       │
└────────────┬────────────────────────────┘
             │
             ├─ Worker Process 1 (PID: 1235)
             ├─ Worker Process 2 (PID: 1236)
             └─ Worker Process 3 (PID: 1237)

┌─────────────────────────────────────────┐
│      Python Flask Application           │
│      (processes requests)               │
│      PID: 2345                          │
└────────────┬────────────────────────────┘
             │
             ├─ Worker 1 (PID: 2346)
             └─ Worker 2 (PID: 2347)

┌─────────────────────────────────────────┐
│      PostgreSQL Database                │
│      (stores data)                      │
│      PID: 3456                          │
└────────────┬────────────────────────────┘
             │
             ├─ Background Writer (PID: 3457)
             ├─ WAL Writer (PID: 3458)
             └─ Stats Collector (PID: 3459)
```

We'll use this example throughout to understand process management.

---

## 📊 **PROCESS STATES (Understanding What Processes Are Doing)**

Every process is in one of these states:

| State | Symbol | Description | Example |
|-------|--------|-------------|---------|
| **Running** | `R` | Currently using CPU | Your Flask app processing a request |
| **Sleeping** | `S` | Waiting for something (I/O, network) | Nginx waiting for HTTP requests |
| **Stopped** | `T` | Paused (Ctrl+Z) | Process you suspended |
| **Zombie** | `Z` | Finished but parent hasn't collected exit status | Orphaned process (BAD) |
| **Uninterruptible Sleep** | `D` | Waiting for disk I/O (can't be interrupted) | Database writing to disk |

---

### **Process State Lifecycle**

```
    ┌─────────┐
    │  START  │
    └────┬────┘
         │
         ▼
    ┌─────────┐
    │ RUNNING │ ◄─────┐
    └────┬────┘       │
         │            │
         ├────────────┘ (gets CPU time)
         │
         ├─────► ┌──────────┐
         │       │ SLEEPING │ (waiting for I/O)
         │       └─────┬────┘
         │             │
         │             └─────► (I/O ready, back to queue)
         │
         ├─────► ┌──────────┐
         │       │ STOPPED  │ (Ctrl+Z)
         │       └─────┬────┘
         │             │
         │             └─────► (SIGCONT to resume)
         │
         └─────► ┌──────────┐
                 │  ZOMBIE  │ (finished, waiting for parent)
                 └─────┬────┘
                       │
                       ▼
                   [ EXIT ]
```

---

## 👁️ **VIEWING PROCESSES (THE BASICS)**

### **ps - Process Status (Snapshot)**

The `ps` command shows currently running processes.

**Basic syntax:**
```bash
ps [OPTIONS]
```

---

#### **Common ps Commands:**

```bash
# Show YOUR processes
ps
# Output:
  PID TTY          TIME CMD
 1234 pts/0    00:00:00 bash
 5678 pts/0    00:00:00 ps

# Show ALL processes (detailed)
ps aux
# a = all users
# u = user-friendly format
# x = include processes without terminal

# Show processes in tree format (hierarchy)
ps auxf
# f = forest (tree view)

# Show specific columns
ps -eo pid,ppid,cmd,%mem,%cpu
# -e = all processes
# -o = custom output format
```

---

#### **Understanding ps aux Output:**

```bash
$ ps aux | head -5
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1 168820 11234 ?        Ss   10:00   0:05 /sbin/init
www-data  1234  0.2  0.5 234567 45678 ?        S    10:05   0:12 nginx: worker
sam       2345  5.0  2.0 567890 89012 ?        Sl   10:10   2:30 python app.py
postgres  3456  1.0  3.5 890123 123456 ?       Ss   10:15   0:45 /usr/lib/postgresql/14/bin/postgres
```

**Column breakdown:**

```
USER     → Owner of the process
PID      → Process ID (unique number)
%CPU     → CPU usage percentage
%MEM     → Memory usage percentage
VSZ      → Virtual memory size (KB)
RSS      → Resident set size (actual physical memory in KB)
TTY      → Terminal (? = no terminal, pts/0 = pseudo-terminal)
STAT     → Process state (R=Running, S=Sleeping, Z=Zombie, etc.)
START    → When process started
TIME     → Total CPU time used
COMMAND  → The command that started the process
```

---

### **Real-World Example: Finding Your Web App**

**Scenario:** Your Flask app is running somewhere. You need to find it.

```bash
# Method 1: Search by name
$ ps aux | grep python
sam  2345  5.0  2.0 567890 89012 ?  Sl  10:10  2:30 python app.py
sam  2346  3.0  1.5 456789 67890 ?  S   10:10  1:45 python app.py
sam  2347  3.0  1.5 456789 67890 ?  S   10:10  1:45 python app.py

# Found it! PID 2345 is the main process
# PIDs 2346, 2347 are worker processes

# Method 2: Find parent-child relationship
$ ps auxf | grep -A 5 python
sam  2345  5.0  2.0 567890 89012 ?  Sl  10:10  2:30 python app.py
sam  2346  3.0  1.5 456789 67890 ?  S   10:10  1:45  \_ python app.py
sam  2347  3.0  1.5 456789 67890 ?  S   10:10  1:45  \_ python app.py
                                                      └─ Child processes (workers)
```

---

### **pgrep - Process Grep (Find PIDs by Name)**

```bash
# Find process ID by name
pgrep python
# Output:
2345
2346
2347

# Show process name and PID
pgrep -a python
# Output:
2345 python app.py
2346 python app.py
2347 python app.py

# Find processes by specific user
pgrep -u sam
# Shows all PIDs owned by user 'sam'

# Count processes
pgrep -c nginx
# Output: 4 (1 master + 3 workers)
```

---

### **top - Real-Time Process Monitoring**

`top` is like `ps` but updates continuously (like Task Manager on Windows).

```bash
top
```

**Output:**
```
top - 15:23:45 up 5:23, 2 users, load average: 0.45, 0.32, 0.28
Tasks: 198 total,   1 running, 197 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.2 us,  1.3 sy,  0.0 ni, 93.2 id,  0.3 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   5923.5 total,   1234.2 free,   3456.3 used,   1233.0 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   2145.2 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 2345 sam       20   0  567890  89012  12345 S   5.0   2.0   2:30.45 python
 3456 postgres  20   0  890123 123456  23456 S   1.0   3.5   0:45.23 postgres
 1234 www-data  20   0  234567  45678   8901 S   0.2   0.5   0:12.34 nginx
```

**Key sections:**

1. **Header (System summary):**
   ```
   load average: 0.45, 0.32, 0.28
                 └─ 1min 5min 15min average
   
   Load average explained:
   - < 1.0 = system not busy
   - 1.0 = system fully utilized
   - > 1.0 = processes waiting for CPU
   ```

2. **Tasks:**
   ```
   198 total, 1 running, 197 sleeping, 0 stopped, 0 zombie
                                                  └─ BAD if not zero!
   ```

3. **CPU usage:**
   ```
   %Cpu(s): 5.2 us, 1.3 sy, 0.0 ni, 93.2 id
            │       │       │       └─ idle (doing nothing)
            │       │       └─ nice (low priority processes)
            │       └─ system (kernel operations)
            └─ user (applications)
   ```

4. **Memory:**
   ```
   MiB Mem: 5923.5 total, 1234.2 free, 3456.3 used
            └─ Total RAM  └─ Available └─ In use
   ```

---

### **Interactive top Commands:**

While top is running, press:

```
h     → Help (show all commands)
q     → Quit
k     → Kill a process (enter PID)
r     → Renice (change priority)
M     → Sort by memory usage
P     → Sort by CPU usage (default)
1     → Show individual CPU cores
u     → Filter by user
```

**Example usage:**

```bash
# Start top
top

# Press 'M' to sort by memory
# Now you see which process uses most RAM

# Press 'k', enter PID 2345
# Type signal: 15 (SIGTERM - graceful shutdown)
```

---

### **htop - Better Alternative to top**

`htop` is like `top` but with:
- ✅ Better colors and UI
- ✅ Mouse support
- ✅ Easier process killing
- ✅ Tree view built-in

```bash
# Install if not present
sudo apt install htop     # Ubuntu/Debian
sudo yum install htop     # RHEL/CentOS

# Run it
htop
```

**htop advantages:**
- F5: Tree view (see parent-child relationships)
- F9: Kill process (menu-driven)
- F6: Sort by column
- F4: Filter by name

---

## 🌳 **PROCESS HIERARCHY (Parent-Child Relationships)**

### **Understanding the Tree**

Every process (except PID 1) has a **parent process**.

```bash
# Show process tree
pstree

# Output (simplified):
systemd(1)─┬─nginx(1234)─┬─nginx(1235)
           │             ├─nginx(1236)
           │             └─nginx(1237)
           ├─python(2345)─┬─python(2346)
           │              └─python(2347)
           ├─postgres(3456)─┬─postgres(3457)
           │                ├─postgres(3458)
           │                └─postgres(3459)
           └─bash(4567)───pstree(4568)
```

**Reading the tree:**
- `systemd(1)` is the root (PID 1) - init system
- `nginx(1234)` is parent of workers (1235, 1236, 1237)
- `bash(4567)` started `pstree(4568)` (your command)

---

### **PPID (Parent Process ID)**

```bash
# Show PID and PPID
$ ps -eo pid,ppid,cmd | grep nginx
 PID  PPID CMD
1234     1 nginx: master process
1235  1234 nginx: worker process
1236  1234 nginx: worker process
1237  1234 nginx: worker process
         └─ PPID 1234 = parent is nginx master
```

**What this tells us:**
- PID 1234 (nginx master) was started by PID 1 (systemd)
- PIDs 1235-1237 (workers) were spawned by PID 1234 (master)

---

### **What Happens When Parent Dies?**

**Scenario 1: Proper cleanup**
```bash
# Kill nginx master
kill 1234

# What happens:
1. Master receives SIGTERM
2. Master tells workers to shutdown
3. Workers exit cleanly
4. Master exits
5. All gone ✓
```

**Scenario 2: Orphaned processes**
```bash
# If parent dies suddenly without cleaning up:
1. Child processes become orphans
2. systemd (PID 1) adopts them
3. Their PPID changes to 1
```

**Scenario 3: Zombie processes** (BAD!)
```bash
# If parent doesn't collect child's exit status:
1. Child finishes execution
2. Child becomes <defunct> (zombie)
3. Shows as 'Z' state in ps
4. Wastes process table entry

# Fix: Kill the parent, systemd will clean up zombies
```

---

## ⚙️ **FOREGROUND VS BACKGROUND PROCESSES**

### **Foreground Process**

Runs in your terminal, blocks your prompt until it finishes.

```bash
# Run a long task (foreground)
$ python app.py
# Your terminal is now blocked
# You can't type other commands
# Press Ctrl+C to stop it
```

**Before:**
```bash
sam@server:~$ python app.py
Starting Flask app...
* Running on http://0.0.0.0:5000
█  ← cursor stuck here, can't type
```

---

### **Background Process**

Runs in the background, returns your prompt immediately.

```bash
# Run in background (add &)
$ python app.py &
[1] 2345
       └─ PID assigned

# Your prompt returns immediately
sam@server:~$ █  ← can type other commands

# Check background jobs
$ jobs
[1]+  Running    python app.py &
```

**After (with &):**
```bash
sam@server:~$ python app.py &
[1] 2345
sam@server:~$ █  ← prompt returned, can work
sam@server:~$ ps aux | grep python
sam  2345  0.1  0.5  ...  python app.py
```

---

### **Job Control Commands**

```bash
# Start a process
$ python app.py

# Press Ctrl+Z (suspend it)
^Z
[1]+  Stopped    python app.py

# Check jobs
$ jobs
[1]+  Stopped    python app.py

# Resume in foreground
$ fg %1
python app.py
# (now running in foreground)

# OR resume in background
$ bg %1
[1]+ python app.py &
# (now running in background)
```

---

### **Real-World Example:**

```bash
# Start multiple background processes
$ nginx &
[1] 1234

$ python app.py &
[2] 2345

$ postgres -D /data &
[3] 3456

# Check all jobs
$ jobs
[1]   Running    nginx &
[2]-  Running    python app.py &
[3]+  Running    postgres -D /data &
                 └─ + means most recent

# Bring python to foreground
$ fg %2
python app.py
# Now in foreground, Ctrl+C will kill it

# Send it back to background
^Z  (Ctrl+Z)
$ bg %2
[2]+ python app.py &
```

---

### **nohup - Run Process Immune to Hangup**

Problem: If you logout, background processes die.

```bash
# WITHOUT nohup
$ python app.py &
# Logout
# Process dies ❌

# WITH nohup
$ nohup python app.py &
nohup: ignoring input and appending output to 'nohup.out'
[1] 2345

# Logout
# Process keeps running ✓

# Output redirected to nohup.out
$ tail -f nohup.out
Starting Flask app...
```

**Production use:**
```bash
# Proper background service
nohup python app.py > /var/log/app.log 2>&1 &
#                     └─ stdout      └─ stderr also to stdout
```

---

## ⚡ **PROCESS SIGNALS (Communicating with Processes)**

### **What Are Signals?**

Signals are **messages sent to processes** to tell them to do something.

**Think of it like:**
- Tapping someone on the shoulder (gentle request)
- Shouting at them (forceful demand)
- Pulling the plug (immediate termination)

---

### **Common Signals**

| Signal | Number | Name | What It Does | Catchable? |
|--------|--------|------|--------------|------------|
| **SIGTERM** | 15 | Terminate | "Please exit gracefully" | ✅ Yes |
| **SIGKILL** | 9 | Kill | "DIE NOW!" (force kill) | ❌ No |
| **SIGHUP** | 1 | Hangup | "Reload config" | ✅ Yes |
| **SIGINT** | 2 | Interrupt | Ctrl+C (keyboard interrupt) | ✅ Yes |
| **SIGSTOP** | 19 | Stop | Pause process (Ctrl+Z) | ❌ No |
| **SIGCONT** | 18 | Continue | Resume paused process | ✅ Yes |
| **SIGUSR1** | 10 | User 1 | Custom signal | ✅ Yes |
| **SIGUSR2** | 12 | User 2 | Custom signal | ✅ Yes |

---

### **Catchable vs Uncatchable**

**Catchable signals** (process can handle them):
```bash
# Process receives SIGTERM
# Process code can:
1. Save work in progress
2. Close database connections
3. Write logs
4. Then exit cleanly
```

**Uncatchable signals** (process MUST obey):
```bash
# Process receives SIGKILL
# Process immediately terminated
# No cleanup possible
# Use only as last resort!
```

---

### **Sending Signals with kill Command**

The `kill` command sends signals (not just kills!).

**Syntax:**
```bash
kill [SIGNAL] PID

# Default signal is SIGTERM (15)
kill 2345              # Send SIGTERM to PID 2345
kill -15 2345          # Same as above
kill -SIGTERM 2345     # Same as above

# Force kill
kill -9 2345           # Send SIGKILL (force)
kill -SIGKILL 2345     # Same as above

# Reload config
kill -HUP 2345         # Send SIGHUP
kill -1 2345           # Same as above
```

---

### **Real-World Examples:**

#### **Example 1: Graceful Shutdown (SIGTERM)**

```bash
# Your Flask app is running
$ ps aux | grep python
sam  2345  5.0  2.0  ...  python app.py

# Send graceful shutdown signal
$ kill 2345            # or kill -15 2345

# What happens inside the app:
# 1. Receives SIGTERM
# 2. Finishes current requests
# 3. Closes database connections
# 4. Writes final logs
# 5. Exits with code 0

# Check if it's gone
$ ps aux | grep 2345
# (no output - process exited cleanly)
```

**BEFORE:**
```
Request processing → SIGTERM received → Finish request → Close DB → Exit
└─ App running        └─ Signal received  └─ Clean shutdown
```

---

#### **Example 2: Force Kill (SIGKILL)**

```bash
# Process is frozen/hanging
$ ps aux | grep python
sam  2345  5.0  2.0  ...  python app.py

# Try graceful first
$ kill 2345
# Wait 5 seconds...
# Process still there?

$ ps aux | grep 2345
sam  2345  5.0  2.0  ...  python app.py
# Still running! It's frozen

# Force kill
$ kill -9 2345

# Check immediately
$ ps aux | grep 2345
# (no output - process killed immediately)
```

**What's the difference?**

```
SIGTERM (kill 2345):
  Process can: ignore, delay, clean up
  May not work: if process is frozen

SIGKILL (kill -9 2345):
  Kernel kills process immediately
  Always works
  BUT: No cleanup! Can leave orphaned resources
```

---

#### **Example 3: Reload Config (SIGHUP)**

```bash
# Nginx is running
$ ps aux | grep nginx
root  1234  0.0  0.1  ...  nginx: master

# You edited /etc/nginx/nginx.conf
# Need to reload without downtime

# Send SIGHUP (reload config)
$ kill -HUP 1234        # or: sudo nginx -s reload

# What happens:
# 1. Master receives SIGHUP
# 2. Reads new config
# 3. Starts new workers with new config
# 4. Gracefully shuts down old workers
# 5. No downtime!

# Verify
$ ps aux | grep nginx
root  1234  0.0  0.1  ...  nginx: master
www-data 8901  ... nginx: worker  ← new PIDs
www-data 8902  ... nginx: worker  ← reloaded!
```

**BEFORE vs AFTER:**

```
BEFORE SIGHUP:
nginx master (1234)
├─ worker (1235) - old config
├─ worker (1236) - old config
└─ worker (1237) - old config

↓ [Send SIGHUP]

AFTER SIGHUP:
nginx master (1234)  ← same PID
├─ worker (8901) - NEW config ← new workers
├─ worker (8902) - NEW config
└─ worker (8903) - NEW config
```

---

### **killall - Kill by Process Name**

```bash
# Kill all processes with name "python"
killall python

# With signal
killall -9 python          # Force kill all
killall -HUP nginx         # Reload all nginx

# Be careful with wildcards!
killall -9 nginx  # Kills ALL nginx processes (master + workers)
```

---

### **pkill - Kill by Pattern**

```bash
# Kill by partial name
pkill python               # Matches "python", "python3", etc.

# Kill by user
pkill -u sam               # Kill all processes owned by sam

# Kill by pattern with signal
pkill -9 -f "app.py"       # Force kill processes matching "app.py"
#      │  └─ match full command line
#      └─ signal 9 (SIGKILL)
```

---

## 🎚️ **PROCESS PRIORITY (nice and renice)**

### **Understanding Priority**

Linux uses **priority values** to decide which process gets CPU time first.

**Priority range:**
```
-20  ← Highest priority (most CPU time)
  0  ← Default priority
+19  ← Lowest priority (least CPU time)
```

**Nice value:**
- Lower number = Higher priority = More CPU
- Higher number = Lower priority = Less CPU (being "nice" to others)

---

### **Viewing Priority**

```bash
# Show nice value (NI column)
$ ps -eo pid,ni,cmd | head -5
  PID  NI CMD
    1   0 /sbin/init
 1234   0 nginx: master
 2345   0 python app.py
 3456  10 backup-script.sh  ← lower priority (nice)
```

---

### **Setting Priority with nice**

```bash
# Start process with lower priority
$ nice -n 10 python backup.py &
#        └─ niceness value (0-19 for regular users)

# Start with higher priority (requires sudo)
$ sudo nice -n -10 python critical-app.py &
#                └─ negative value = higher priority
```

---

### **Real-World Example: Background Backup**

**Problem:** Backup script consumes too much CPU, slowing down web app.

**WITHOUT nice:**
```bash
$ python backup.py &

# Check CPU usage
$ top
  PID USER      NI  %CPU  %MEM  COMMAND
 2345 sam        0  45.0   2.0  python app.py       ← web app
 5678 sam        0  50.0   1.0  python backup.py    ← backup hogging CPU!
```

**WITH nice:**
```bash
$ nice -n 19 python backup.py &
#          └─ lowest priority

# Check CPU usage now
$ top
  PID USER      NI  %CPU  %MEM  COMMAND
 2345 sam        0  85.0   2.0  python app.py       ← web app gets most CPU
 5678 sam       19  10.0   1.0  python backup.py    ← backup uses leftover CPU
```

**What changed?**
- Backup still runs, but doesn't steal CPU from web app
- Web app responsive for users
- Backup completes when system is idle

---

### **Changing Priority with renice**

```bash
# Process already running with default priority
$ ps -eo pid,ni,cmd | grep backup
5678   0 python backup.py

# Change its priority (make it nicer)
$ renice -n 15 -p 5678
5678 (process ID) old priority 0, new priority 15

# Verify
$ ps -eo pid,ni,cmd | grep backup
5678  15 python backup.py  ← priority changed!
```

---

### **Production Scenario: Database Maintenance**

```bash
# Critical web app running
$ ps -eo pid,ni,cmd | grep app
2345   0 python app.py

# Need to run database maintenance
$ sudo nice -n 15 pg_dump mydb > backup.sql &
[1] 6789

# Web app priority stays high (0)
# Backup runs with low priority (15)
# Users don't notice performance impact
```

---

## �️ **PRACTICAL PRODUCTION SCENARIOS**

### **Scenario 1: "The System Is Slow!"**

**Symptoms:** Users complaining, website loading slowly.

**Step 1: Check load average**
```bash
$ uptime
15:45:01 up 2 days, 5:32, 2 users, load average: 8.45, 7.32, 5.28
                                                    └─ WAY too high for 4-core CPU!
```

**Step 2: Find CPU hog**
```bash
$ top
# Press 'P' to sort by CPU
  PID USER      %CPU  %MEM  COMMAND
 5678 sam       95.0   1.0  python rogue-script.py  ← Found it!
 2345 sam        2.0   2.0  python app.py
 1234 www-data   0.5   0.5  nginx
```

**Step 3: Investigate**
```bash
$ ps -fp 5678
UID  PID  PPID  C STIME TTY TIME     CMD
sam  5678 4567 95 15:30  ?  00:15:23 python rogue-script.py

# Check what it's doing
$ strace -p 5678  # Advanced: trace system calls
# (shows it's in infinite loop)
```

**Step 4: Kill it**
```bash
# Try graceful first
$ kill 5678

# Wait 5 seconds...still there?
$ ps -p 5678
# Still running!

# Force kill
$ kill -9 5678

# Verify
$ ps -p 5678
# (no output - killed)

# Check load
$ uptime
15:46:01 up 2 days, 5:33, 2 users, load average: 2.15, 5.32, 4.28
                                                    └─ Dropping!
```

**BEFORE vs AFTER:**

```
BEFORE:
- Load: 8.45 (system overloaded)
- Rogue script: 95% CPU
- Web app: 2% CPU (starved)
- Users: "Site is slow!" ❌

AFTER (kill -9 rogue script):
- Load: 2.15 (normal)
- Web app: 45% CPU (healthy)
- Users: "Site fast again!" ✅
```

---

### **Scenario 2: "Process Won't Die!"**

**Problem:** Trying to kill process, but it won't exit.

```bash
# Try to kill
$ kill 2345
# Wait...

$ ps -p 2345
  PID TTY      STAT   TIME COMMAND
 2345 ?        D      5:23 python app.py
              └─ 'D' state = Uninterruptible sleep (usually disk I/O)
```

**What's happening:**
- Process waiting for disk I/O
- Cannot be interrupted (even by SIGTERM)
- Common causes: NFS mount hung, disk failure

**Solutions:**

```bash
# 1. Wait for I/O to complete (if temporary)
# (patience...)

# 2. Force kill (usually doesn't work for 'D' state)
$ kill -9 2345
# Still in 'D' state

# 3. Fix underlying issue
# Check disk
$ dmesg | grep -i error
# Check mounts
$ mount | grep nfs

# 4. Reboot (last resort)
$ sudo reboot
```

---

### **Scenario 3: "Zombie Processes!"**

```bash
# Check for zombies
$ ps aux | grep 'Z'
sam  3456  0.0  0.0     0     0 ?        Z    10:30   0:00 [python] <defunct>
sam  3457  0.0  0.0     0     0 ?        Z    10:30   0:00 [python] <defunct>
                                         └─ Zombie state!
```

**What are zombies?**
- Process finished execution
- Parent didn't collect exit status
- Wastes process table entry

**How to fix:**

```bash
# Find parent
$ ps -eo pid,ppid,stat,cmd | grep 3456
3456  2345  Z  [python] <defunct>
      └─ parent PID

# Kill the parent
$ kill 2345

# Parent exits, systemd adopts zombies and cleans them
$ ps aux | grep 'Z'
# (no output - zombies gone!)
```

**Prevention:** Write programs that properly wait for child processes.

---

### **Scenario 4: "Graceful Rolling Restart"**

**Scenario:** Update Flask app without downtime.

```bash
# Current state
$ ps aux | grep python
sam  2345  0.1  2.0  ...  python app.py  ← master
sam  2346  0.1  1.5  ...  python app.py  ← worker 1
sam  2347  0.1  1.5  ...  python app.py  ← worker 2

# Step 1: Start new version
$ python app-v2.py &
[1] 9001

$ ps aux | grep python
sam  2345  0.1  2.0  ...  python app.py      ← old version
sam  2346  0.1  1.5  ...  python app.py
sam  2347  0.1  1.5  ...  python app.py
sam  9001  0.1  2.0  ...  python app-v2.py  ← new version

# Step 2: Gracefully shutdown old workers
$ kill 2346   # worker 1
$ kill 2347   # worker 2
# (they finish current requests, then exit)

# Step 3: Shutdown old master
$ kill 2345

# Final state
$ ps aux | grep python
sam  9001  0.1  2.0  ...  python app-v2.py  ← only new version running!
```

---

## 🧪 **MINI-TASK: Process Hunting & Management**

**Objective:** Practice finding, monitoring, and controlling processes in a realistic scenario.

**Scenario:** You'll simulate a busy system and practice process management.

---

### **Tasks:**

1. **Start Multiple Background Processes**
   ```bash
   # Create a CPU-intensive script
   cat > ~/cpu-hog.sh << 'EOF'
   #!/bin/bash
   while true; do
     echo "Computing..." > /dev/null
   done
   EOF
   
   chmod +x ~/cpu-hog.sh
   
   # Start 3 copies in background
   ~/cpu-hog.sh &
   ~/cpu-hog.sh &
   nice -n 10 ~/cpu-hog.sh &
   ```

2. **Monitor and Document**
   - Use `ps aux` to find all 3 processes
   - Use `top` to see their CPU usage
   - Use `pgrep` to find PIDs by name
   - Note which one has lower CPU usage (hint: nice value)

3. **Process Control**
   - Suspend one process (Ctrl+Z)
   - Resume it in background
   - Change priority of one process using `renice`
   - Kill all processes gracefully (SIGTERM)
   - If any refuse to die, force kill (SIGKILL)

4. **Create Report:** `/home/sam/DevopsRoadmap/Phase-1-Linux/process-report.txt`
   - PIDs of all 3 processes
   - Their nice values
   - CPU usage of each
   - Commands used to kill them
   - Screenshot or paste of `top` output

**Security Challenge:**
- What happens if you `kill -9 1` (init process)?
- Why do zombie processes exist?
- When should you use SIGTERM vs SIGKILL?

**Expected Time:** 45-60 minutes

**Deliverable:** Show me your `process-report.txt` and explain what you learned.

---

## 📌 **Quick Reference Card**

```bash
# Viewing Processes
ps aux                      # All processes detailed
ps auxf                     # Process tree
pgrep python                # Find PID by name
pstree                      # Visual tree
top                         # Real-time monitor
htop                        # Better top

# Background Jobs
command &                   # Run in background
jobs                        # List background jobs
fg %1                       # Bring job 1 to foreground
bg %1                       # Resume job 1 in background
Ctrl+Z                      # Suspend current process
nohup command &             # Run immune to hangup

# Signals
kill PID                    # Graceful shutdown (SIGTERM)
kill -9 PID                 # Force kill (SIGKILL)
kill -HUP PID               # Reload config (SIGHUP)
killall process-name        # Kill all by name
pkill pattern               # Kill by pattern

# Priority
nice -n 10 command          # Start with low priority
renice -n 15 -p PID         # Change running process priority

# Information
ps -eo pid,ppid,ni,cmd      # Custom columns
ps -fp PID                  # Full info for specific PID
pgrep -a python             # Show PIDs and full command
```

---
**Complete this mini-task, then we'll move to Module 4: System Services (systemd).**

---

## 📝 **LEARNING LOG (Post Mini-Task)**

**Date:** January 23, 2026  
**Task Score:** 92/100

### **Doubt 1: What happens with `kill -9 1`?**

**My understanding:** System may crash.

**Correction:** The kernel **PROTECTS PID 1**:
- PID 1 (init/systemd) is the ancestor of ALL processes
- Linux kernel **ignores** SIGKILL sent to PID 1
- Even `sudo kill -9 1` returns silently with no effect
- If PID 1 somehow dies, kernel panics → system crash
- This is a safety mechanism, not a vulnerability

```bash
$ sudo kill -9 1
# Nothing happens - kernel protects init
```

---

### **Doubt 2: Zombie vs Orphan Processes (Important!)**

**My confusion:** I thought zombies were abandoned processes that keep running.

**Correction:** That describes ORPHANS, not ZOMBIES!

| Aspect | ZOMBIE | ORPHAN |
|--------|--------|--------|
| Parent status | Alive (not calling wait()) | Dead |
| Child status | **Finished** (defunct) | Still running |
| Process state | Z | R/S (normal) |
| Resource usage | None (just PID entry) | Normal |
| Can be killed? | No (already dead) | Yes |

**ZOMBIE lifecycle:**
```
Parent spawns child → Child FINISHES → Parent hasn't read exit status
                                       └─ Child = ZOMBIE (waiting to be reaped)
```

**Why zombies exist:**
- Unix design: parent must collect child's exit status
- Child waits in zombie state until parent calls `wait()`
- Takes no resources, just occupies a PID slot

**How to fix zombies:**
```bash
# Find zombie's parent
$ ps -eo pid,ppid,stat,cmd | grep Z
3456  2345  Z  [python] <defunct>
      └─ Parent PID

# Kill parent → init adopts → cleans up zombie
$ kill 2345
```

**ORPHAN lifecycle:**
```
Parent spawns child → Parent DIES → init/systemd adopts child
                                    └─ Child keeps running normally
```

---

### **Doubt 3: SIGTERM vs SIGKILL Best Practice**

**Confirmed understanding:**
```bash
# Always try graceful first
kill PID              # SIGTERM - process can clean up
sleep 5               # Wait for graceful shutdown

# Only force if necessary
ps -p PID && kill -9 PID   # SIGKILL only if still alive
```

**Why this matters in production:**
- SIGTERM allows DB connections to close properly
- SIGTERM allows logs to flush
- SIGKILL can leave orphaned resources, corrupted files

---

### **Key Takeaways from Module 3:**

1. **Process = program in execution** with PID, state, priority
2. **`ps aux`** for snapshot, **`top/htop`** for real-time monitoring
3. **Load average** should be ≤ number of CPU cores
4. **Signals** are how we communicate with processes
5. **nice/renice** control CPU priority (lower = more CPU)
6. **`/proc/[PID]/`** exposes all process information
7. **`ulimit`** controls resource limits (watch for "too many open files")
8. **Zombie ≠ Orphan** — zombies are finished, orphans are still running
