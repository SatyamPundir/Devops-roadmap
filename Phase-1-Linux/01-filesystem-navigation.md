# Module 1: File System & Navigation

## 🎯 Why This Matters in Production

When a service crashes at 3 AM, you need to know:
- Where are the logs? (`/var/log/`)
- Where's the config? (`/etc/`)
- Where's the binary? (`/usr/bin/`)
- Where are temporary files? (`/tmp/`)

**No time to Google. This must be muscle memory.**

---

## 📁 Linux Filesystem Hierarchy Standard (FHS)

Linux organizes everything as files in a **single tree structure**, starting from root (`/`). There are no drive letters (C:, D:) like Windows.

### **Critical Directories You MUST Know**

| Directory | Purpose | Production Relevance |
|-----------|---------|---------------------|
| `/` | Root directory - everything starts here | - |
| `/bin` | Essential user binaries (ls, cat, cp) | Basic commands always available |
| `/sbin` | System binaries (requires root) | System administration tools |
| `/etc` | Configuration files | **All service configs live here** |
| `/home` | User home directories | User data, personal configs |
| `/root` | Root user's home | Admin's home directory |
| `/var` | Variable data (logs, caches, databases) | **Logs in /var/log, app data** |
| `/tmp` | Temporary files (cleared on reboot) | Scratch space, uploads |
| `/usr` | User programs and data | Installed software |
| `/usr/bin` | User binaries | Most commands you use |
| `/usr/local` | Locally installed software | Custom/manual installs |
| `/opt` | Optional application packages | Third-party software |
| `/proc` | Virtual filesystem for processes | **Runtime system info** |
| `/sys` | Virtual filesystem for kernel/devices | **Hardware & driver info** |
| `/dev` | Device files | Disks, terminals, null device |
| `/mnt` | Temporary mount points | Manual mounts |
| `/media` | Removable media mount points | USB drives, CD-ROMs |
| `/lib` | Essential shared libraries | System libraries |
| `/boot` | Boot loader files (kernel) | System boot files |

### **Production Examples**

```bash
# Where's the nginx config?
/etc/nginx/nginx.conf

# Where are nginx logs?
/var/log/nginx/access.log
/var/log/nginx/error.log

# Where's the Docker daemon config?
/etc/docker/daemon.json

# Where are Docker logs?
/var/log/docker.log
# OR
journalctl -u docker  # systemd logs

# Where's the application deployed?
/opt/myapp/          # Third-party apps
/srv/www/            # Web content (alternative)
/var/www/html/       # Apache/nginx default web root

# Where are cron job logs?
/var/log/cron
/var/log/syslog      # On Debian/Ubuntu

# Where are system logs?
/var/log/syslog      # Ubuntu/Debian
/var/log/messages    # RHEL/CentOS
```

---

## 🧭 Navigation Commands

### **pwd** - Print Working Directory
Shows where you are currently.

```bash
pwd
# Output: /home/sam/DevopsRoadmap
```

### **cd** - Change Directory

```bash
cd /var/log                  # Absolute path
cd nginx                     # Relative path (go into nginx subdir)
cd ..                        # Go up one level
cd ../..                     # Go up two levels
cd ~                         # Go to your home directory
cd -                         # Go to previous directory
cd                           # Also goes to home (shortcut)
```

**Production Tip**: Use absolute paths in scripts to avoid ambiguity.

```bash
# BAD (depends on current directory)
cd logs && tail -f app.log

# GOOD (explicit)
cd /var/log/myapp && tail -f app.log

# BETTER (no cd needed)
tail -f /var/log/myapp/app.log
```

### **ls** - List Directory Contents

```bash
ls                           # List files in current directory
ls -l                        # Long format (permissions, owner, size, date)
ls -la                       # Include hidden files (start with .)
ls -lh                       # Human-readable sizes (KB, MB, GB)
ls -lt                       # Sort by modification time (newest first)
ls -ltr                      # Sort by time (oldest first)
ls -lS                       # Sort by size
ls -R                        # Recursive (show subdirectories)
ls /var/log                  # List specific directory
```

**Production Example**:
```bash
# Find the most recently modified log file
ls -lt /var/log/*.log | head -n 5

# Find largest files in /var
ls -lhS /var/log | head -n 10

# Check if hidden config files exist
ls -la ~/.ssh/
```

### **tree** - Visualize Directory Structure

```bash
tree                         # Show tree of current directory
tree -L 2                    # Limit depth to 2 levels
tree -d                      # Directories only
tree -a                      # Include hidden files
```

*Note: May need to install: `sudo apt install tree` or `sudo yum install tree`*

---

## 📍 Path Concepts

### **Absolute Path**
Starts from root (`/`). Always works regardless of current directory.

```bash
/home/sam/DevopsRoadmap/Phase-1-Linux
```

### **Relative Path**
Relative to current directory.

```bash
# If you're in /home/sam
DevopsRoadmap/Phase-1-Linux

# Using . (current directory)
./script.sh

# Using .. (parent directory)
../Documents/
```

### **Special Path Symbols**

| Symbol | Meaning |
|--------|---------|
| `/` | Root directory |
| `~` | Home directory (`/home/username`) |
| `.` | Current directory |
| `..` | Parent directory |
| `-` | Previous directory |

---

## 🔗 Links (Symbolic & Hard Links)

### **Symbolic Link (Symlink)**
A pointer/shortcut to another file or directory.

```bash
# Create symlink
ln -s /var/log/nginx/access.log ~/nginx-access.log

# Now you can read logs from home directory
tail ~/nginx-access.log

# Check where symlink points to
ls -l ~/nginx-access.log
# Output: nginx-access.log -> /var/log/nginx/access.log
```

**Production Use Case**:
```bash
# Common pattern: point "current" to latest deployment
ln -sf /opt/myapp/releases/v1.2.3 /opt/myapp/current

# Rollback? Just change the symlink
ln -sf /opt/myapp/releases/v1.2.2 /opt/myapp/current
```

### **Hard Link**
Multiple filenames for the same data on disk.

```bash
ln /path/to/original /path/to/hardlink
```

**Key Differences**:
- Symlink: Can link to directories, can cross filesystems, breaks if target deleted
- Hard link: Cannot link directories, same filesystem only, survives target deletion

**Production Note**: Symlinks are more common in DevOps workflows.

---

## 🔍 File Inspection

### **file** - Determine File Type

```bash
file /bin/ls                    # ELF executable
file /etc/hosts                 # ASCII text
file /tmp/image.jpg             # JPEG image
```

### **stat** - Detailed File Information

```bash
stat /etc/passwd
# Shows: size, permissions, timestamps (access, modify, change), inode
```

### **which** - Locate Command Binary

```bash
which python                    # /usr/bin/python
which docker                    # /usr/bin/docker
```

**Production Use**: Verify which binary is being executed (important when multiple versions exist).

### **whereis** - Locate Binary, Source, and Man Page

```bash
whereis nginx
# nginx: /usr/sbin/nginx /etc/nginx /usr/share/nginx /usr/share/man/man8/nginx.8.gz
```

---

## 🎯 Hidden Files & Dotfiles

Files starting with `.` are hidden.

```bash
ls -a              # Show hidden files
ls -la ~/.ssh/     # Common hidden directory (SSH keys)
ls -la ~/.bashrc   # Shell configuration file
```

**Important Hidden Configs**:
- `~/.bashrc` - Bash shell configuration
- `~/.ssh/` - SSH keys and config
- `~/.config/` - Application configs
- `~/.kube/config` - Kubernetes config
- `~/.aws/` - AWS CLI config

---

## ⚠️ Common Mistakes & Pitfalls

### **Mistake 1: Not Understanding Root `/`**

```bash
# WRONG: Will try to cd to /root (root user's home)
cd root

# CORRECT: Root filesystem
cd /
```

### **Mistake 2: Spaces in Paths**

```bash
# WRONG: Tries to cd to "My" directory
cd My Documents

# CORRECT: Use quotes or escape
cd "My Documents"
cd My\ Documents
```

### **Mistake 3: Relative vs Absolute Paths in Scripts**

```bash
# WRONG: Breaks if script run from different directory
cd logs

# CORRECT: Use absolute path or resolve it
cd /var/log/myapp
# OR
cd "$(dirname "$0")/logs"  # Relative to script location
```

### **Mistake 4: Following Symlinks Blindly**

```bash
# Check if it's a symlink before deleting
ls -l /opt/myapp/current    # Is it a symlink?
rm /opt/myapp/current       # Safe (removes link, not target)
rm -rf /opt/myapp/current/  # DANGEROUS (follows link and deletes target!)
```

---

## 🛠️ Production Debugging Scenarios

### **Scenario 1: "Where are the logs?"**

```bash
# Start with common log directories
ls -lt /var/log/

# Application-specific
ls -lt /var/log/nginx/
ls -lt /var/log/myapp/

# Check systemd logs
journalctl -u nginx -n 50
```

### **Scenario 2: "Where is this service configured?"**

```bash
# Check /etc
ls /etc/nginx/
ls /etc/systemd/system/

# Find config files
find /etc -name "*nginx*"
```

### **Scenario 3: "Disk full - what's eating space?"**

```bash
# Check disk usage
df -h

# Find large directories in /var
du -sh /var/* | sort -hr | head -10

# Find large files
find /var -type f -size +100M
```

---

## 🧪 **MINI-TASK: Filesystem Exploration**

**Objective**: Explore your system's filesystem and understand its structure.

**Tasks**:

1. **Create a report file**: `/home/sam/DevopsRoadmap/Phase-1-Linux/filesystem-report.txt`

2. **Document the following** (using only terminal commands):

   a) List the top 5 largest directories in `/var`  
   b) Find all files in `/etc` that contain "ssh" in their name  
   c) Identify what type of file `/bin/bash` is  
   d) Create a symlink in your home directory called `logs` that points to `/var/log`  
   e) List all hidden files in your home directory  
   f) Show the full path of where the `python3` command is located  
   g) Count total number of files in `/usr/bin/`

3. **Include the commands you used** in the report file

**Expected Time**: 30-45 minutes

**Deliverable**: Show me the contents of your `filesystem-report.txt` file.

---

## 📌 Quick Reference Card

```bash
# Navigation
pwd                         # Where am I?
cd /path                    # Go somewhere
cd ..                       # Go up
cd ~                        # Go home
cd -                        # Go back

# Listing
ls -la                      # List all (detailed)
ls -lh                      # Human-readable sizes
ls -lt                      # Sort by time

# Finding
which command               # Where's the binary?
find /path -name "*.log"    # Find files by name
tree -L 2                   # Visualize structure

# Links
ln -s target link           # Create symlink
ls -l file                  # Check if symlink

# File info
file filename               # What type?
stat filename               # Detailed info
```

---

## Doubts:
- I wanted to go the /root directory but access was denied, how to traverse there?
- How can i only list directories but till level 2 ?
- was not able to understand the symlink vs hardlink, is hardlink like copy pasting the file?

## Difficulty Faced:
- permission denied for the listing and finding in the VAR directory.

---
**Complete this mini-task, then we'll move to Module 2: File Operations & Permissions.**
