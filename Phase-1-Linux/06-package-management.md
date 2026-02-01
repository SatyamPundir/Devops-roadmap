# Module 6: Package Management

## 🎯 Why This Matters in Production

**Scenario**: Security alert! A critical vulnerability (CVE-2026-XXXX) affects OpenSSL. Your entire fleet of 50 servers needs patching within 24 hours.

You need to instantly know:
- **How do I check what version is installed?**
- **How do I update a specific package?**
- **How do I update ALL packages safely?**
- **How do I rollback if something breaks?**

**In production, package management = system security + stability.**

---

## 🧠 **PACKAGE MANAGEMENT FUNDAMENTALS**

### **What is a Package?**

A **package** is a bundled collection of:

```
┌─────────────────────────────────────────────────────────────┐
│                      PACKAGE CONTENTS                        │
├─────────────────────────────────────────────────────────────┤
│  📁 Executable binaries     /usr/bin/nginx                  │
│  📁 Libraries               /usr/lib/x86_64-linux-gnu/...   │
│  📁 Configuration files     /etc/nginx/nginx.conf           │
│  📁 Documentation           /usr/share/doc/nginx/           │
│  📁 Man pages               /usr/share/man/man8/nginx.8.gz  │
│  📋 Metadata                Version, dependencies, scripts  │
│  📜 Pre/post install scripts                                │
└─────────────────────────────────────────────────────────────┘
```

---

### **Package Managers**

Different Linux distributions use different package managers:

| Distribution | Package Format | Low-Level Tool | High-Level Tool |
|--------------|---------------|----------------|-----------------|
| **Debian/Ubuntu** | `.deb` | `dpkg` | `apt` |
| **RHEL/CentOS/Fedora** | `.rpm` | `rpm` | `dnf`/`yum` |
| **Arch Linux** | `.pkg.tar.zst` | `pacman` | `pacman` |
| **Alpine** | `.apk` | `apk` | `apk` |

**This module focuses on Debian/Ubuntu (apt)** — the most common in cloud environments.

---

### **Low-Level vs High-Level Tools**

```
┌─────────────────────────────────────────────────────────────┐
│                    HIGH-LEVEL: apt                           │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ • Resolves dependencies automatically                   ││
│  │ • Downloads from repositories                           ││
│  │ • Handles upgrades intelligently                        ││
│  │ • User-friendly interface                               ││
│  └─────────────────────────────────────────────────────────┘│
│                          ↓                                   │
│                    LOW-LEVEL: dpkg                           │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ • Installs/removes individual .deb files                ││
│  │ • No dependency resolution                              ││
│  │ • No network access                                     ││
│  │ • Direct package manipulation                           ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

**Rule of thumb:** Use `apt` for everything. Use `dpkg` only for troubleshooting or local `.deb` files.

---

### **Repositories**

Packages come from **repositories** — online servers hosting packages.

```bash
# View configured repositories
$ cat /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu noble main restricted
deb http://archive.ubuntu.com/ubuntu noble-updates main restricted
deb http://security.ubuntu.com/ubuntu noble-security main restricted

# Additional repositories
$ ls /etc/apt/sources.list.d/
docker.list
nodejs.list
```

**Repository components:**

| Component | Contents |
|-----------|----------|
| `main` | Officially supported free software |
| `restricted` | Proprietary drivers (NVIDIA, etc.) |
| `universe` | Community-maintained free software |
| `multiverse` | Non-free software |

---

### 🎯 **Running Example: Web Application Stack**

Throughout this module, we'll manage packages for our **production stack**:

```
┌─────────────────────────────────────────────────────────────┐
│                    PACKAGE DEPENDENCIES                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  nginx (web server)                                          │
│  ├─ depends: libc6, libpcre3, libssl3, zlib1g               │
│  └─ recommends: nginx-common                                 │
│                                                              │
│  python3 (runtime)                                           │
│  ├─ depends: libc6, libpython3-stdlib                       │
│  └─ recommends: python3-pip                                  │
│                                                              │
│  postgresql (database)                                       │
│  ├─ depends: libc6, libssl3, postgresql-common              │
│  └─ recommends: postgresql-client                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚡ **APT - THE MAIN TOOL**

### **Essential apt Commands**

| Command | Purpose | When To Use |
|---------|---------|-------------|
| `apt update` | Refresh package lists | Before any install/upgrade |
| `apt upgrade` | Upgrade installed packages | Regular maintenance |
| `apt full-upgrade` | Upgrade with dependency changes | Major updates |
| `apt install` | Install packages | Adding new software |
| `apt remove` | Remove packages (keep config) | Uninstalling |
| `apt purge` | Remove packages + config | Complete removal |
| `apt autoremove` | Remove unused dependencies | Cleanup |
| `apt search` | Search for packages | Finding software |
| `apt show` | Show package details | Research before install |
| `apt list` | List packages | Inventory |

---

## 🔄 **UPDATING THE SYSTEM**

### **Step 1: Update Package Lists**

```bash
$ sudo apt update
Hit:1 http://archive.ubuntu.com/ubuntu noble InRelease
Get:2 http://security.ubuntu.com/ubuntu noble-security InRelease [126 kB]
Get:3 http://archive.ubuntu.com/ubuntu noble-updates InRelease [126 kB]
Fetched 252 kB in 2s (126 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
45 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

**What this does:**
- Downloads latest package lists from repositories
- Does NOT install anything
- Must run before install/upgrade

---

### **Step 2: Upgrade Packages**

#### **Standard Upgrade (Safe)**

```bash
$ sudo apt upgrade
Reading package lists... Done
Building dependency tree... Done
Calculating upgrade... Done
The following packages will be upgraded:
  base-files curl libcurl4 openssl libssl3
5 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Need to get 2,345 kB of archives.
Do you want to continue? [Y/n] Y
```

**What `upgrade` does:**
- Upgrades packages to newer versions
- Never removes packages
- Never installs new dependencies
- Safe for production

---

#### **Full Upgrade (More Aggressive)**

```bash
$ sudo apt full-upgrade
The following packages will be REMOVED:
  old-package
The following NEW packages will be installed:
  new-dependency
The following packages will be upgraded:
  nginx postgresql
```

**What `full-upgrade` does:**
- May remove packages if needed
- May install new dependencies
- Use for major version upgrades
- Review changes carefully!

---

### **BEFORE vs AFTER Update**

```bash
# BEFORE
$ apt list --installed | grep openssl
openssl/noble,now 3.0.2-0ubuntu1.6 amd64 [installed]

# After update + upgrade
$ apt list --installed | grep openssl
openssl/noble-security,now 3.0.2-0ubuntu1.7 amd64 [installed]
                                        ^^^ version bumped
```

---

### **Production Update Strategy**

```bash
# 1. Check what will be upgraded
$ sudo apt update
$ apt list --upgradable

# 2. Review the changes (especially for critical packages)
$ apt changelog openssl  # View what changed

# 3. Upgrade in maintenance window
$ sudo apt upgrade -y

# 4. Verify services still work
$ systemctl status nginx postgresql myapp

# 5. If issues, check logs
$ journalctl -u nginx --since "10 minutes ago"
```

---

## 📦 **INSTALLING PACKAGES**

### **Basic Installation**

```bash
$ sudo apt install nginx
Reading package lists... Done
Building dependency tree... Done
The following additional packages will be installed:
  nginx-common nginx-core
Suggested packages:
  fcgiwrap nginx-doc
The following NEW packages will be installed:
  nginx nginx-common nginx-core
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
Need to get 615 kB of archives.
Do you want to continue? [Y/n] Y
```

---

### **Install Specific Version**

```bash
# List available versions
$ apt list -a nginx
nginx/noble-updates 1.24.0-2ubuntu1.1 amd64
nginx/noble 1.24.0-2ubuntu1 amd64

# Install specific version
$ sudo apt install nginx=1.24.0-2ubuntu1
```

---

### **Install Multiple Packages**

```bash
$ sudo apt install nginx postgresql python3-pip
```

---

### **Install Without Prompts (Scripting)**

```bash
$ sudo apt install -y nginx
# -y = assume yes to all prompts
```

---

### **Install from .deb File**

```bash
# Download a .deb file
$ wget https://example.com/package.deb

# Install with apt (handles dependencies!)
$ sudo apt install ./package.deb

# Or with dpkg (no dependency resolution)
$ sudo dpkg -i package.deb
$ sudo apt install -f  # Fix missing dependencies
```

---

### **Simulate Installation (Dry Run)**

```bash
$ apt install --simulate nginx
# OR
$ apt install -s nginx

# Shows what WOULD happen without doing it
Inst nginx-common (1.24.0-2ubuntu1 Ubuntu:24.04/noble [all])
Inst nginx-core (1.24.0-2ubuntu1 Ubuntu:24.04/noble [amd64])
Inst nginx (1.24.0-2ubuntu1 Ubuntu:24.04/noble [all])
```

---

## 🗑️ **REMOVING PACKAGES**

### **remove vs purge**

```bash
# Remove package, KEEP configuration files
$ sudo apt remove nginx
# Config files remain in /etc/nginx/

# Remove package AND configuration files
$ sudo apt purge nginx
# Everything gone!
```

**When to use which:**

| Scenario | Use |
|----------|-----|
| Temporary removal, might reinstall | `remove` |
| Complete cleanup | `purge` |
| Troubleshooting (fresh config) | `purge` then reinstall |

---

### **Clean Up Unused Dependencies**

```bash
# When you install package A, it installs dependencies B and C
# When you remove A, B and C remain as orphans

# Find and remove orphaned packages
$ sudo apt autoremove

# More aggressive cleanup
$ sudo apt autoremove --purge
```

---

### **Production Example: Complete Package Removal**

```bash
# Scenario: Removing old software completely

# Step 1: Stop the service
$ sudo systemctl stop old-app

# Step 2: Disable from boot
$ sudo systemctl disable old-app

# Step 3: Purge the package
$ sudo apt purge old-app

# Step 4: Remove orphaned dependencies
$ sudo apt autoremove --purge

# Step 5: Verify removal
$ which old-app
# (no output = removed)
$ ls /etc/old-app/
# ls: cannot access '/etc/old-app/': No such file or directory
```

---

## 🔍 **SEARCHING AND INFORMATION**

### **Search for Packages**

```bash
# Search by name/description
$ apt search nginx
Sorting... Done
Full Text Search... Done
nginx/noble 1.24.0-2ubuntu1 all
  small, powerful, scalable web/proxy server

nginx-extras/noble 1.24.0-2ubuntu1 amd64
  nginx web server with extra modules

# Search for exact name
$ apt search --names-only ^nginx$
```

---

### **Show Package Details**

```bash
$ apt show nginx
Package: nginx
Version: 1.24.0-2ubuntu1
Priority: optional
Section: httpd
Maintainer: Ubuntu Developers
Installed-Size: 48.1 kB
Depends: nginx-core | nginx-extras
Homepage: https://nginx.org
Description: small, powerful, scalable web/proxy server
```

---

### **List Installed Packages**

```bash
# All installed packages
$ apt list --installed

# Count installed packages
$ apt list --installed | wc -l
587

# Filter by name
$ apt list --installed | grep nginx
nginx/noble,now 1.24.0-2ubuntu1 all [installed]
nginx-common/noble,now 1.24.0-2ubuntu1 all [installed,automatic]
nginx-core/noble,now 1.24.0-2ubuntu1 amd64 [installed,automatic]
```

---

### **Check Package Dependencies**

```bash
# What does nginx need?
$ apt depends nginx
nginx
  Depends: nginx-core | nginx-extras
  
# What needs nginx? (reverse dependencies)
$ apt rdepends nginx
nginx
Reverse Depends:
  nginx-extras
  nginx-core
```

---

### **Which Package Provides a File?**

```bash
# Find which package owns a file
$ dpkg -S /usr/sbin/nginx
nginx-core: /usr/sbin/nginx

# Search for packages providing a file (even not installed)
$ apt-file search /usr/bin/htop
htop: /usr/bin/htop

# (requires: sudo apt install apt-file && sudo apt-file update)
```

---

### **List Files in a Package**

```bash
# Files from installed package
$ dpkg -L nginx-core
/.
/usr
/usr/sbin
/usr/sbin/nginx
/usr/share
/usr/share/doc
...

# Files from .deb file (not installed)
$ dpkg -c package.deb
```

---

## 🏷️ **HOLDING PACKAGES (Prevent Updates)**

### **Why Hold Packages?**

Sometimes you need to **freeze** a package version:
- Known working version, don't want surprises
- Compatibility requirements
- Waiting for bug fix in newer version

---

### **Hold a Package**

```bash
# Prevent nginx from being upgraded
$ sudo apt-mark hold nginx
nginx set on hold.

# Verify
$ apt-mark showhold
nginx

# Check status
$ apt list nginx
nginx/noble,now 1.24.0-2ubuntu1 all [installed,upgradable to: 1.24.0-2ubuntu1.1]
# Note: Won't be upgraded due to hold
```

---

### **Unhold a Package**

```bash
$ sudo apt-mark unhold nginx
Canceled hold on nginx.
```

---

### **Production Example: Holding Critical Package**

```bash
# Scenario: Your app is tested with nginx 1.24.0, don't upgrade until tested

$ sudo apt-mark hold nginx nginx-common nginx-core

# Later, when ready to upgrade
$ sudo apt-mark unhold nginx nginx-common nginx-core
$ sudo apt upgrade nginx
```

---

## 🔧 **DPKG - LOW-LEVEL OPERATIONS**

### **When to Use dpkg**

- Installing local `.deb` files
- Troubleshooting broken packages
- Querying package database
- Extracting packages without installing

---

### **Essential dpkg Commands**

```bash
# Install .deb file
$ sudo dpkg -i package.deb

# Remove package
$ sudo dpkg -r package-name

# Purge package (remove + config)
$ sudo dpkg -P package-name

# List installed packages
$ dpkg -l

# Search installed packages
$ dpkg -l | grep nginx

# Show package info
$ dpkg -s nginx

# List files in installed package
$ dpkg -L nginx-core

# Find which package owns file
$ dpkg -S /usr/sbin/nginx
```

---

### **Fix Broken Packages**

```bash
# Scenario: dpkg -i failed due to missing dependencies

$ sudo dpkg -i custom-app.deb
dpkg: dependency problems prevent configuration of custom-app:
 custom-app depends on libssl3; however:
  Package libssl3 is not installed.

# Fix missing dependencies
$ sudo apt install -f
# OR
$ sudo apt --fix-broken install

# This downloads and installs the missing dependencies
```

---

### **Reconfigure a Package**

```bash
# Re-run post-install configuration
$ sudo dpkg-reconfigure tzdata
# Opens timezone selection dialog

$ sudo dpkg-reconfigure locales
# Opens locale selection
```

---

## 🗂️ **MANAGING REPOSITORIES**

### **Add a Repository**

#### **Method 1: add-apt-repository (Easiest)**

```bash
# Add PPA (Personal Package Archive)
$ sudo add-apt-repository ppa:deadsnakes/ppa

# Add official third-party repo
$ sudo add-apt-repository "deb https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Update after adding
$ sudo apt update
```

#### **Method 2: Manual (More Control)**

```bash
# Create sources list file
$ sudo nano /etc/apt/sources.list.d/docker.list

# Add:
deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu noble stable
```

---

### **Add GPG Key (Security)**

Repositories are signed with GPG keys for security:

```bash
# Modern method (recommended)
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Verify
$ ls /usr/share/keyrings/
docker-archive-keyring.gpg
```

---

### **Remove a Repository**

```bash
# Remove with add-apt-repository
$ sudo add-apt-repository --remove ppa:deadsnakes/ppa

# Or delete the file
$ sudo rm /etc/apt/sources.list.d/docker.list

# Update
$ sudo apt update
```

---

### **Real-World Example: Installing Docker**

```bash
# 1. Install prerequisites
$ sudo apt update
$ sudo apt install ca-certificates curl gnupg

# 2. Add Docker's GPG key
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 3. Add Docker repository
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Install Docker
$ sudo apt update
$ sudo apt install docker-ce docker-ce-cli containerd.io

# 5. Verify
$ docker --version
Docker version 24.0.7, build afdd53b
```

---

## 🧹 **CLEANING UP**

### **Remove Downloaded Package Files**

```bash
# apt downloads .deb files to /var/cache/apt/archives/

# Check cache size
$ du -sh /var/cache/apt/archives/
245M    /var/cache/apt/archives/

# Remove old packages (keeps current versions)
$ sudo apt autoclean

# Remove ALL cached packages
$ sudo apt clean

# Verify
$ du -sh /var/cache/apt/archives/
8.0K    /var/cache/apt/archives/
```

---

### **Complete System Cleanup**

```bash
# Remove orphaned packages
$ sudo apt autoremove --purge

# Clean package cache
$ sudo apt clean

# Remove old kernels (keep current + one previous)
$ sudo apt autoremove --purge

# Check disk space saved
$ df -h /
```

---

## 🛠️ **PRACTICAL PRODUCTION SCENARIOS**

### **Scenario 1: Security Update**

```bash
# CVE announced for openssl
# Step 1: Check current version
$ dpkg -l | grep openssl
ii  openssl  3.0.2-0ubuntu1.6  amd64  ...

# Step 2: Update package lists
$ sudo apt update

# Step 3: Check if update available
$ apt list --upgradable | grep openssl
openssl/noble-security 3.0.2-0ubuntu1.7 amd64 [upgradable from: 3.0.2-0ubuntu1.6]

# Step 4: View changelog
$ apt changelog openssl | head -20
# Shows CVE fixes

# Step 5: Upgrade
$ sudo apt install openssl
# OR for just this package:
$ sudo apt install --only-upgrade openssl

# Step 6: Verify
$ openssl version
OpenSSL 3.0.2 7 Feb 2026 (Library: OpenSSL 3.0.2)

# Step 7: Restart affected services
$ sudo systemctl restart nginx postgresql
```

---

### **Scenario 2: Package Won't Install**

```bash
# Error: broken dependencies
$ sudo apt install custom-app
The following packages have unmet dependencies:
 custom-app : Depends: libfoo (>= 2.0) but 1.0 is to be installed

# Solution 1: Try fix-broken
$ sudo apt --fix-broken install

# Solution 2: Install specific dependency version
$ sudo apt install libfoo=2.0

# Solution 3: Simulate to see what's happening
$ apt install -s custom-app

# Solution 4: Check held packages
$ apt-mark showhold

# Solution 5: Force (last resort, dangerous!)
$ sudo dpkg --force-depends -i custom-app.deb
```

---

### **Scenario 3: Rollback After Bad Update**

```bash
# Nginx broken after update!

# Step 1: Check what version was installed
$ grep nginx /var/log/dpkg.log
2026-02-01 10:00:00 upgrade nginx:amd64 1.24.0-2ubuntu1 1.24.0-2ubuntu1.1

# Step 2: Downgrade to previous version
$ sudo apt install nginx=1.24.0-2ubuntu1

# Step 3: Hold to prevent re-upgrade
$ sudo apt-mark hold nginx

# Step 4: Verify service works
$ sudo systemctl restart nginx
$ curl http://localhost/
```

---

### **Scenario 4: Find What Changed**

```bash
# Something broke, what was recently installed/upgraded?

# Check dpkg log
$ grep " install \| upgrade " /var/log/dpkg.log | tail -20
2026-02-01 10:00:00 upgrade nginx:amd64 1.24.0-2ubuntu1 1.24.0-2ubuntu1.1
2026-02-01 10:00:01 install new-package:amd64 <none> 1.0.0

# Check apt history
$ cat /var/log/apt/history.log | tail -50
Start-Date: 2026-02-01  10:00:00
Commandline: apt upgrade -y
Upgrade: nginx:amd64 (1.24.0-2ubuntu1, 1.24.0-2ubuntu1.1)
End-Date: 2026-02-01  10:00:05
```

---

## 🧪 **MINI-TASK: Package Management Operations**

### **Objective:**
Practice essential package management tasks in a production-like scenario.

---

### **Part 1: System Update**

```bash
# 1. Update package lists
# 2. Check how many packages can be upgraded
# 3. List the upgradable packages
# 4. Perform the upgrade
```

---

### **Part 2: Package Installation**

```bash
# 1. Search for the 'htop' package
# 2. Show details about htop
# 3. Install htop
# 4. Verify installation by running htop
# 5. Find which package provides /usr/bin/htop
# 6. List all files installed by htop
```

---

### **Part 3: Package Information**

```bash
# 1. List all installed packages containing 'python'
# 2. Check what packages nginx depends on
# 3. Check what version of openssl is installed
# 4. Find the changelog for a recent update
```

---

### **Part 4: Package Holds**

```bash
# 1. Put a hold on the 'nginx' package
# 2. Verify the hold is in place
# 3. Remove the hold
```

---

### **Part 5: Cleanup**

```bash
# 1. Remove a package you installed (htop)
# 2. Clean the package cache
# 3. Remove orphaned packages
# 4. Check cache size before and after
```

---

### **Part 6: Create Report**

Create `/home/sam/DevopsRoadmap/Phase-1-Linux/package-report.txt` containing:

1. Number of packages that were upgradable
2. Output of `apt show htop`
3. Output of `dpkg -L htop` (first 20 lines)
4. Output of `apt-mark showhold` (showing nginx held)
5. Cache size before and after `apt clean`

---

### **Security Questions:**

1. Why should you run `apt update` before `apt install`?
2. What's the difference between `apt remove` and `apt purge`?
3. Why would you hold a package?
4. What does `apt autoremove` do and when should you run it?
5. Why do repositories need GPG keys?

---

**Expected Time:** 45-60 minutes

**Deliverable:** Show me your `package-report.txt` and explain what you learned.

---

## 📌 **Quick Reference Card**

```bash
# Update & Upgrade
sudo apt update                  # Refresh package lists
sudo apt upgrade                 # Upgrade packages (safe)
sudo apt full-upgrade            # Upgrade with removals allowed
apt list --upgradable            # Show what can be upgraded

# Install & Remove
sudo apt install nginx           # Install package
sudo apt install nginx=1.24.0    # Install specific version
sudo apt install ./package.deb   # Install local .deb
sudo apt remove nginx            # Remove (keep config)
sudo apt purge nginx             # Remove + delete config
sudo apt autoremove              # Remove orphaned packages

# Search & Info
apt search nginx                 # Search packages
apt show nginx                   # Package details
apt depends nginx                # Show dependencies
apt list --installed             # List installed
dpkg -L nginx                    # Files in package
dpkg -S /usr/sbin/nginx          # Which package owns file

# Hold/Unhold
sudo apt-mark hold nginx         # Prevent upgrades
sudo apt-mark unhold nginx       # Allow upgrades
apt-mark showhold                # List held packages

# Cleanup
sudo apt clean                   # Remove all cached .debs
sudo apt autoclean               # Remove old cached .debs
sudo apt autoremove --purge      # Remove orphans + config

# Fix Problems
sudo apt --fix-broken install    # Fix dependency issues
sudo dpkg --configure -a         # Configure pending packages
sudo dpkg-reconfigure package    # Reconfigure package

# Logs
/var/log/dpkg.log               # Package operations log
/var/log/apt/history.log        # apt command history
```

---

**Complete this mini-task, then we'll move to Module 7: Text Processing.**
