# Module 2: File Operations & Permissions

## 🎯 Why This Matters in Production

**Scenario**: You deploy an application. It crashes with "Permission denied."

You need to instantly know:
- Who owns the file?
- What permissions does it have?
- How do I fix it **safely** without breaking security?

**Wrong permissions = security breach or broken deployment. Both get you fired.**

---

## 📝 File Operations (CRUD)

### **Create Files & Directories**

```bash
# Create empty file
touch file.txt
touch /tmp/test.log

# Create multiple files
touch file1.txt file2.txt file3.txt

# Create directory
mkdir mydir

# Create nested directories
mkdir -p /path/to/nested/dir
mkdir -p ~/projects/{frontend,backend,devops}

# Create file with content
echo "Hello" > file.txt              # Overwrite
echo "World" >> file.txt             # Append
cat > file.txt << EOF                # Multi-line (press Ctrl+D when done)
Line 1
Line 2
EOF
```

**Production Example**:
```bash
# Setup application directory structure
mkdir -p /opt/myapp/{bin,config,logs,data}
```

### **Read/View Files**

```bash
cat file.txt                          # Display entire file
cat -n file.txt                       # With line numbers

head file.txt                         # First 10 lines
head -n 20 file.txt                   # First 20 lines

tail file.txt                         # Last 10 lines
tail -n 50 /var/log/nginx/error.log   # Last 50 lines
tail -f /var/log/app.log              # Follow (live updates) - CRITICAL for debugging

less file.txt                         # Paginated view (q to quit)
more file.txt                         # Similar to less (less features)

# Search while viewing in less
# Press / then type search term, press n for next match
```

**Production Use**:
```bash
# Watch live logs during deployment
tail -f /var/log/myapp/app.log

# Check last 100 lines of error log
tail -n 100 /var/log/nginx/error.log

# View large log file
less +G /var/log/syslog              # +G starts at end
```

### **Update/Edit Files**

```bash
# Command-line editors
nano file.txt                         # Beginner-friendly (Ctrl+X to exit)
vim file.txt                          # Advanced (press i to insert, :wq to save&quit)
vi file.txt                           # Older version of vim

# Redirect to update
echo "New content" > file.txt         # Replace entire file
echo "Append this" >> file.txt        # Add to end

# Inline substitution (sed)
sed -i 's/old/new/g' file.txt         # Replace all occurrences
```

**Production Example**:
```bash
# Update config file
sudo sed -i 's/DEBUG=True/DEBUG=False/' /opt/myapp/config/settings.py

# Append to config
echo "listen 8080;" | sudo tee -a /etc/nginx/nginx.conf
```

### **Copy Files**

```bash
cp source.txt dest.txt                # Copy file
cp file.txt /tmp/                     # Copy to directory
cp -r mydir/ /backup/mydir/           # Copy directory recursively
cp -p file.txt backup.txt             # Preserve permissions & timestamps
cp -v source dest                     # Verbose (show progress)

# Copy multiple files
cp file1.txt file2.txt /destination/
```

**Production Example**:
```bash
# Backup config before editing
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
sudo cp -p /etc/nginx/nginx.conf /etc/nginx/nginx.conf.$(date +%Y%m%d)

# Copy application for deployment
cp -r /tmp/myapp-v1.2/* /opt/myapp/
```

### **Move/Rename Files**

```bash
mv old.txt new.txt                    # Rename
mv file.txt /tmp/                     # Move
mv *.log /var/log/archive/            # Move multiple files
mv -i source dest                     # Interactive (ask before overwrite)
```

**Production Example**:
```bash
# Rotate logs
mv /var/log/app.log /var/log/app.log.1

# Deploy new version
mv /tmp/app-v2 /opt/myapp/releases/v2
```

### **Delete Files**

```bash
rm file.txt                           # Delete file
rm -i file.txt                        # Interactive (ask confirmation)
rm -f file.txt                        # Force (no confirmation)
rm -r mydir/                          # Delete directory recursively
rm -rf mydir/                         # Force recursive delete - DANGEROUS!

# Delete empty directory
rmdir emptydir/
```

**⚠️ CRITICAL WARNING**:

```bash
# NEVER run these (can destroy entire system)
sudo rm -rf /                         # Deletes EVERYTHING
sudo rm -rf /*                        # Also deletes everything
rm -rf /var /log                      # Typo can be catastrophic

# Always double-check before running rm -rf
```

**Production Safe Practice**:
```bash
# 1. Use ls first to verify
ls -la /tmp/old_logs/*
# 2. Then delete
rm -rf /tmp/old_logs/*

# Or use trash/move to temp instead
mv /tmp/old_logs /tmp/backup_$(date +%s)
```

---

## 🔐 Linux Permission Model (COMPREHENSIVE GUIDE)

### 📖 **The Foundation: Understanding the 3-Layer Security Model**

Linux uses a **3-layer permission system** to control who can do what with files and directories.

**Every single file/directory has THREE groups of permissions:**

```
┌─────────────────────────────────────────────┐
│         Who can access this file?           │
├─────────────────────────────────────────────┤
│  1. OWNER   (the user who owns the file)    │
│  2. GROUP   (users in the file's group)     │
│  3. OTHERS  (everyone else on the system)   │
└─────────────────────────────────────────────┘
```

**Each group can have THREE types of permissions:**

```
┌─────────────────────────────────────────────┐
│        What can they do with it?            │
├─────────────────────────────────────────────┤
│  r = READ    (view/list contents)           │
│  w = WRITE   (modify/delete)                │
│  x = EXECUTE (run as program/enter dir)     │
└─────────────────────────────────────────────┘
```

---

### 🎯 **Running Example: A Simple Web Application**

Let's use a **real scenario** throughout this entire section. Imagine you're deploying a web application:

```
/var/www/myapp/
├── app.py           (Python application - needs to be executed)
├── config.ini       (Configuration - readable but not executable)
├── secrets.key      (API keys - must be private!)
└── uploads/         (Directory where users upload files)
```

We'll use this example to understand every concept below.

---

### 📊 **Understanding `ls -l` Output (Step-by-Step)**

Let's run `ls -l` on our example web app:

```bash
$ ls -l /var/www/myapp/
```

**Output:**
```
-rwxr-xr-x 1 sam  developers 2048 Jan 10 10:00 app.py
-rw-r--r-- 1 sam  developers  512 Jan 10 10:00 config.ini
-rw------- 1 sam  developers  256 Jan 10 10:00 secrets.key
drwxrwxr-x 2 sam  developers 4096 Jan 10 10:00 uploads
```

Let me break down **ONE LINE** in extreme detail:

```
-rwxr-xr-x 1 sam developers 2048 Jan 10 10:00 app.py
```

**Here's what EACH part means:**

```
Position:  1    2-4   5-7   8-10  11  12    13       14   15      16    17
           │    │     │     │     │   │     │        │    │       │     │
           ▼    ▼     ▼     ▼     ▼   ▼     ▼        ▼    ▼       ▼     ▼
           -   rwx   r-x   r-x    1   sam   developers 2048 Jan 10 10:00 app.py
           │    │     │     │     │   │     │        │    │       │     │
           │    │     │     │     │   │     │        │    │       │     └─ Filename
           │    │     │     │     │   │     │        │    │       └─ Time
           │    │     │     │     │   │     │        │    └─ Date
           │    │     │     │     │   │     │        └─ Size (bytes)
           │    │     │     │     │   │     └─ Group name
           │    │     │     │     │   └─ Owner name
           │    │     │     │     └─ Number of hard links
           │    │     │     └─ Permissions for OTHERS
           │    │     └─ Permissions for GROUP
           │    └─ Permissions for OWNER
           └─ File type
```

**Let's understand each section:**

---

#### **Section 1: File Type (Position 1)**

The **first character** tells you WHAT KIND of thing this is:

| Symbol | Type | Example |
|--------|------|---------|
| **-** | Regular file | `-rw-r--r--` (text file, binary, script) |
| **d** | Directory | `drwxr-xr-x` (folder) |
| **l** | Symbolic link | `lrwxrwxrwx` (shortcut/pointer) |
| **c** | Character device | `crw-rw-rw-` (keyboard, mouse) |
| **b** | Block device | `brw-rw----` (hard disk) |
| **s** | Socket | `srwxrwxrwx` (network connection) |
| **p** | Named pipe | `prw-r--r--` (inter-process communication) |

**In our example:**
```
-rwxr-xr-x  app.py         → Regular file (-)
drwxrwxr-x  uploads/       → Directory (d)
```

---

#### **Section 2-4: Permissions (Positions 2-10)**

The next **9 characters** are divided into **3 groups of 3**:

```
-rwxr-xr-x
 │││││││││
 ││││││└└└─── OTHERS permissions (positions 8-10)
 │││└└└────── GROUP permissions (positions 5-7)
 └└└───────── OWNER permissions (positions 2-4)
```

**Each group of 3 follows the pattern: rwx**

```
Position:  1  2  3
           ─  ─  ─
           r  w  x
           │  │  │
           │  │  └─ Execute (can run/enter)
           │  └──── Write (can modify)
           └─────── Read (can view)
```

**If a permission is DENIED, it shows as `-` instead of the letter.**

---

#### **Let's Decode Our Example Files:**

**File 1: app.py**
```
-rwxr-xr-x
 rwx r-x r-x
 │││ │││ │││
 │││ │││ └└└─ OTHERS: r-x (read + execute, no write)
 │││ └└└───── GROUP: r-x (read + execute, no write)
 └└└───────── OWNER: rwx (read + write + execute - FULL control)
```

**Translation:**
- **Owner (sam)**: Can read, modify, and run the file ✅✅✅
- **Group (developers)**: Can read and run, but NOT modify ✅❌✅
- **Others (everyone else)**: Can read and run, but NOT modify ✅❌✅

**Why these permissions?**
- It's an application script that needs to be executed (x)
- Only the owner should modify the code (w only for owner)
- Others can run it but not change it (security)

---

**File 2: config.ini**
```
-rw-r--r--
 rw- r-- r--
 │││ │││ │││
 │││ │││ └└└─ OTHERS: r-- (read only)
 │││ └└└───── GROUP: r-- (read only)
 └└└───────── OWNER: rw- (read + write)
```

**Translation:**
- **Owner (sam)**: Can read and modify ✅✅❌
- **Group (developers)**: Can only read ✅❌❌
- **Others**: Can only read ✅❌❌

**Why these permissions?**
- It's a config file, not a program (no execute needed)
- Owner can update settings (w for owner)
- Others can view settings but not change them (read-only)

---

**File 3: secrets.key**
```
-rw-------
 rw- --- ---
 │││ │││ │││
 │││ │││ └└└─ OTHERS: --- (NO ACCESS AT ALL)
 │││ └└└───── GROUP: --- (NO ACCESS AT ALL)
 └└└───────── OWNER: rw- (read + write only)
```

**Translation:**
- **Owner (sam)**: Can read and modify ✅✅❌
- **Group**: NO ACCESS ❌❌❌
- **Others**: NO ACCESS ❌❌❌

**Why these permissions?**
- Contains sensitive API keys/passwords
- ONLY the owner should see this (security!)
- This is how SSH private keys should be protected

---

**Directory: uploads/**
```
drwxrwxr-x
 rwx rwx r-x
 │││ │││ │││
 │││ │││ └└└─ OTHERS: r-x (can list and enter directory)
 │││ └└└───── GROUP: rwx (full access)
 └└└───────── OWNER: rwx (full access)
```

**Translation:**
- **Owner (sam)**: Full control - list, add, delete files ✅✅✅
- **Group (developers)**: Full control - list, add, delete files ✅✅✅
- **Others**: Can list and enter, but not create/delete files ✅❌✅

---

### 🔢 **Numeric Permissions (Octal Notation)**

**Instead of writing `rwxr-xr-x`, we can write `755`**

Each permission has a **numeric value:**

```
r = 4  (Read)
w = 2  (Write)
x = 1  (Execute)
- = 0  (No permission)
```

**To calculate the number for each group, ADD the values:**

```
rwx = 4 + 2 + 1 = 7  (Full access)
rw- = 4 + 2 + 0 = 6  (Read + Write)
r-x = 4 + 0 + 1 = 5  (Read + Execute)
r-- = 4 + 0 + 0 = 4  (Read only)
-wx = 0 + 2 + 1 = 3  (Write + Execute, no read - rare)
-w- = 0 + 2 + 0 = 2  (Write only - rare)
--x = 0 + 0 + 1 = 1  (Execute only - rare)
--- = 0 + 0 + 0 = 0  (No access)
```

---

### 📝 **Converting Our Example Files to Numeric:**

**File: app.py → -rwxr-xr-x**

```
Step 1: Split into 3 groups
  rwx  |  r-x  |  r-x

Step 2: Calculate each group
  Owner:  rwx = 4+2+1 = 7
  Group:  r-x = 4+0+1 = 5
  Others: r-x = 4+0+1 = 5

Step 3: Combine
  755
```

So **`rwxr-xr-x` = `755`**

---

**File: config.ini → -rw-r--r--**

```
Step 1: Split
  rw-  |  r--  |  r--

Step 2: Calculate
  Owner:  rw- = 4+2+0 = 6
  Group:  r-- = 4+0+0 = 4
  Others: r-- = 4+0+0 = 4

Step 3: Combine
  644
```

So **`rw-r--r--` = `644`**

---

**File: secrets.key → -rw-------**

```
Step 1: Split
  rw-  |  ---  |  ---

Step 2: Calculate
  Owner:  rw- = 4+2+0 = 6
  Group:  --- = 0+0+0 = 0
  Others: --- = 0+0+0 = 0

Step 3: Combine
  600
```

So **`rw-------` = `600`**

---

**Directory: uploads/ → drwxrwxr-x**

```
Step 1: Split (ignore the 'd' - it's just the type)
  rwx  |  rwx  |  r-x

Step 2: Calculate
  Owner:  rwx = 4+2+1 = 7
  Group:  rwx = 4+2+1 = 7
  Others: r-x = 4+0+1 = 5

Step 3: Combine
  775
```

So **`rwxrwxr-x` = `775`**

---

### 📋 **Quick Reference: Common Permission Patterns**

| Numeric | Symbolic | Who Can Do What | Common Use |
|---------|----------|-----------------|------------|
| **755** | `rwxr-xr-x` | Owner: all; Others: read+execute | Scripts, programs, directories |
| **644** | `rw-r--r--` | Owner: read+write; Others: read only | Config files, documents, HTML |
| **600** | `rw-------` | Owner only: read+write | SSH keys, password files, secrets |
| **700** | `rwx------` | Owner only: full access | Private directories, sensitive folders |
| **775** | `rwxrwxr-x` | Owner+Group: all; Others: read+execute | Shared project directories |
| **664** | `rw-rw-r--` | Owner+Group: read+write; Others: read | Shared documents |
| **777** | `rwxrwxrwx` | Everyone: everything | ⚠️ DANGER - Never use! |
| **666** | `rw-rw-rw-` | Everyone: read+write | ⚠️ DANGER - Avoid! |

---

### 🎯 **Permission Meaning: Files vs Directories**

**The SAME permission means DIFFERENT things for files and directories:**

| Permission | For FILES | For DIRECTORIES |
|------------|-----------|-----------------|
| **r (read)** | View file contents<br>`cat file.txt` | List directory contents<br>`ls /mydir/` |
| **w (write)** | Modify/delete file<br>`echo "text" >> file.txt` | Create/delete files inside<br>`rm /mydir/file.txt` |
| **x (execute)** | Run as program<br>`./script.sh` | Enter directory<br>`cd /mydir/` |

---

### 💡 **Example: Why Directory Permissions Matter**

Let's say you have this directory:

```bash
$ ls -ld /var/www/myapp/uploads/
drw-r--r-- 2 sam developers 4096 Jan 10 10:00 uploads/
```

**What can you do?**

```
Owner (sam): rw- = read + write, NO execute
  ✅ Can list files: ls uploads/
  ✅ Can create files: touch uploads/newfile
  ❌ CANNOT enter: cd uploads/  (need x permission!)
```

**This is a common mistake!** Without **x** on a directory, you can't `cd` into it!

**Fix:**
```bash
chmod 755 uploads/  # Now: rwxr-xr-x
cd uploads/  # Works now!
```

---

### 🔍 **Understanding Owner and Group (Positions 12-13)**

Going back to our `ls -l` output:

```
-rwxr-xr-x 1 sam developers 2048 Jan 10 10:00 app.py
              │   │
              │   └─ Group (which group owns this)
              └───── Owner (which user owns this)
```

**Every file has:**
- **One owner** (a user)
- **One group** (a group of users)

**In our example:**
- **Owner:** `sam` (the user who created/owns the file)
- **Group:** `developers` (a group that may contain multiple users)

**Who falls into which category?**

```
┌────────────────────────────────────────┐
│ User "sam" (the owner)                 │
│ → Uses OWNER permissions (rwx)         │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ Users in "developers" group            │
│ (except sam - owner takes precedence)  │
│ → Uses GROUP permissions (r-x)         │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ Everyone else on the system            │
│ → Uses OTHERS permissions (r-x)        │
└────────────────────────────────────────┘
```

**Important:** A user can only be in ONE category. Linux checks in this order:
1. Are you the owner? → Use owner permissions
2. Are you in the group? → Use group permissions
3. Otherwise → Use others permissions

---

### 📊 **Summary Table: Our Web App Example**

| File | Permissions | Numeric | Owner | Group | Why? |
|------|-------------|---------|-------|-------|------|
| `app.py` | `-rwxr-xr-x` | 755 | sam | developers | Executable script; owner can edit |
| `config.ini` | `-rw-r--r--` | 644 | sam | developers | Config file; owner can edit, others read |
| `secrets.key` | `-rw-------` | 600 | sam | developers | Private keys; owner only |
| `uploads/` | `drwxrwxr-x` | 775 | sam | developers | Shared directory; team can add files |

---

### 👥 **UNDERSTANDING USERS & GROUPS (THE BASICS)**

Before we change ownership, you MUST understand how users and groups work in Linux.

---

#### **What Are Users?**

Every person (or service) that accesses the system has a **user account**.

**Types of users:**
1. **Regular users** - Normal people (you, your colleagues)
2. **Root user** - Superuser with ALL permissions (UID 0)
3. **System users** - Used by services (nginx, postgres, docker, etc.)

**Finding Information About Users:**

```bash
# Who am I logged in as?
whoami
# Output: sam

# What's my user ID (UID)?
id -u
# Output: 1000

# Complete info about current user
id
# Output: uid=1000(sam) gid=1000(sam) groups=1000(sam),27(sudo),999(docker)
```

**User information is stored in `/etc/passwd`:**

```bash
cat /etc/passwd | grep sam
# sam:x:1000:1000:Sam:/home/sam:/bin/bash
#  │  │  │    │    │      │         │
#  │  │  │    │    │      │         └─ Default shell
#  │  │  │    │    │      └─ Home directory
#  │  │  │    │    └─ Full name/description
#  │  │  │    └─ Primary group ID (GID)
#  │  │  └─ User ID (UID)
#  │  └─ Password (x means it's in /etc/shadow)
#  └─ Username
```

---

#### **What Are Groups?**

A **group** is a collection of users. Groups make it easy to give permissions to multiple users at once.

**Why groups matter:**
- Instead of giving permissions to 10 developers individually, create a "developers" group
- Add all 10 users to that group
- Now you can give permissions to the entire group at once

**Finding Information About Groups:**

```bash
# List all groups I belong to
groups
# Output: sam sudo docker developers

# List all groups on the system
cat /etc/group
# Or just get group names
cut -d: -f1 /etc/group | sort

# Get info about a specific group
getent group developers
# Output: developers:x:1001:sam,alice,bob
#          │         │  │   └─ Members
#          │         │  └─ GID
#          │         └─ Password (usually x)
#          └─ Group name

# See detailed info about current user's groups
id
# Output: uid=1000(sam) gid=1000(sam) groups=1000(sam),27(sudo),999(docker),1001(developers)
```

**Group information is stored in `/etc/group`:**

```bash
cat /etc/group | grep developers
# developers:x:1001:sam,alice,bob
#     │      │  │    └─ Members of this group
#     │      │  └─ Group ID (GID)
#     │      └─ Password (x means no password)
#     └─ Group name
```

---

#### **Common System Groups You'll See:**

| Group Name | Purpose | Why It Matters |
|------------|---------|----------------|
| `root` | Root/admin group | Full system access |
| `sudo` | Can use sudo command | Run commands as root |
| `wheel` | Admin group (RHEL/CentOS) | Same as sudo |
| `docker` | Can use Docker | Run docker commands without sudo |
| `www-data` | Web server user/group | Nginx/Apache runs as this |
| `adm` | View system logs | Can read `/var/log/` |
| `systemd-journal` | Access systemd logs | Can use `journalctl` |

---

#### **Practical Example: Checking Permissions for Your Web App**

```bash
# Check your current user and groups
$ id
uid=1000(sam) gid=1000(sam) groups=1000(sam),27(sudo),1001(developers)

# This means:
# - Your username: sam
# - Your primary group: sam (GID 1000)
# - You're also in: sudo (can use sudo), developers (team group)

# Check file ownership
$ ls -l /var/www/myapp/app.py
-rwxr-xr-x 1 sam developers 2048 Jan 10 10:00 app.py
              │   │
              │   └─ Group: developers (anyone in developers group gets r-x)
              └───── Owner: sam (gets rwx permissions)

# Who can access this file?
# 1. sam (owner) → rwx (read, write, execute)
# 2. alice, bob (in developers group) → r-x (read, execute)
# 3. Everyone else → r-x (read, execute)
```

---

#### **Creating Groups (Basic Commands)**

```bash
# Create a new group
sudo groupadd webteam

# Add a user to a group
sudo usermod -aG webteam sam
#              │  │
#              │  └─ Group name
#              └──── Append (don't remove from other groups)

# Verify user was added
groups sam
# Output: sam sudo docker developers webteam

# Remove user from a group
sudo gpasswd -d sam webteam
#              │  │   │
#              │  │   └─ Group
#              │  └───── Username
#              └──────── Delete from group

# Delete a group
sudo groupdel webteam
```

---

#### **Important Notes About Groups:**

1. **Primary vs Secondary Groups:**
   - **Primary group**: Your default group (usually same as username)
   - **Secondary groups**: Additional groups you belong to

2. **Group changes require logout:**
   ```bash
   # After adding yourself to a group, you must logout/login
   # or use:
   newgrp groupname  # Opens new shell with that group
   ```

3. **Check effective group:**
   ```bash
   # When you create a file, it gets your current effective group
   touch testfile
   ls -l testfile
   # -rw-r--r-- 1 sam sam 0 Jan 10 12:00 testfile
   #                    └─ Your primary group
   ```

---

## 👤 **CHANGING OWNERSHIP (COMPREHENSIVE GUIDE)**

Now that you understand users and groups, let's learn how to change file ownership.

---

### 🎯 **Why Change Ownership?**

**Common scenarios:**
1. You deploy an app as root, but it should run as `www-data`
2. You copy files from another user's directory
3. You want a team to collaboratively work on files
4. A service needs to own its own files (docker, nginx, postgres)

---

### 🔧 **The chown Command (Change Owner)**

**Syntax:**
```bash
chown [OPTION] USER[:GROUP] FILE
```

**Forms you'll use:**
```bash
chown user file          # Change owner only
chown user:group file    # Change owner AND group
chown :group file        # Change group only (owner stays same)
chown -R user:group dir/ # Recursive (entire directory tree)
```

---

### 📝 **Example 1: Changing File Owner**

**Scenario:** You created files as `sam`, but nginx web server needs to read them. Nginx runs as user `www-data`.

**BEFORE:**
```bash
$ ls -l /var/www/myapp/app.py
-rwxr-xr-x 1 sam sam 2048 Jan 10 10:00 app.py
              │   │
              └───└─ Owner: sam, Group: sam

$ whoami
sam

# Can sam access it? YES (owner has rwx)
$ cat app.py
# ✅ Works!

# Can www-data access it? Let's check
$ sudo -u www-data cat app.py
# ✅ Works! (because others have r-x permission)

# But what if we had permissions: -rwx------
# Then ONLY sam could access it, www-data couldn't!
```

**Let's change ownership to www-data:**

```bash
$ sudo chown www-data app.py
```

**AFTER:**
```bash
$ ls -l app.py
-rwxr-xr-x 1 www-data sam 2048 Jan 10 10:00 app.py
              │        │
              └────────└─ Owner: www-data, Group: sam (unchanged)

# Now www-data is the OWNER, gets rwx permissions
# sam is just in "others" category now, gets r-x only

# Can www-data modify it? YES (owner has rwx)
$ sudo -u www-data echo "test" >> app.py
# ✅ Works!

# Can sam modify it? NO (others only have r-x, no write)
$ echo "test" >> app.py
# ❌ Permission denied
```

**What changed?**
- **Before**: sam = owner → rwx permissions
- **After**: www-data = owner → rwx permissions, sam demoted to "others" → only r-x

---

### 📝 **Example 2: Changing Owner AND Group**

**Scenario:** Your entire team (developers group) should collaboratively own project files.

**BEFORE:**
```bash
$ ls -l /var/www/myapp/config.ini
-rw-r--r-- 1 sam sam 512 Jan 10 10:00 config.ini
              │   │
              └───└─ Owner: sam, Group: sam

# Who can edit it?
# - sam (owner, has rw-)
# - sam group members (only sam) (have r--)
# - Others (have r--)
# Result: ONLY sam can edit
```

**Change owner to sam, group to developers:**

```bash
$ sudo chown sam:developers config.ini
#              │   │
#              │   └─ New group: developers
#              └───── New owner: sam
```

**AFTER:**
```bash
$ ls -l config.ini
-rw-r--r-- 1 sam developers 512 Jan 10 10:00 config.ini
              │   │
              └───└─ Owner: sam, Group: developers

# Check who's in developers group
$ getent group developers
developers:x:1001:sam,alice,bob

# Who can edit it now?
# - sam (owner, has rw-) ✅ Can edit
# - alice, bob (in developers group, have r--) ❌ Can only read

# But wait! We want the team to edit it!
# We need to change GROUP permissions too:
$ chmod 664 config.ini
#         │││
#         ││└─ Others: r-- (read)
#         │└── Group: rw- (read+write)
#         └─── Owner: rw- (read+write)
```

**AFTER PERMISSION CHANGE:**
```bash
$ ls -l config.ini
-rw-rw-r-- 1 sam developers 512 Jan 10 10:00 config.ini
   ││
   │└─ Group now has write permission!
   └── Owner has write permission

# Now BOTH sam AND alice/bob can edit it!
$ sudo -u alice echo "# Added by alice" >> config.ini
# ✅ Works! alice is in developers group with rw- permission
```

---

### 📝 **Example 3: Recursive Ownership Change**

**Scenario:** You deployed an entire application directory, all files need proper ownership.

**BEFORE:**
```bash
$ ls -la /opt/webapp/
drwxr-xr-x 5 root root 4096 Jan 10 10:00 .
-rwxr-xr-x 1 root root 2048 Jan 10 10:00 app.py
-rw-r--r-- 1 root root  512 Jan 10 10:00 config.ini
drwxr-xr-x 2 root root 4096 Jan 10 10:00 logs/
-rw-r--r-- 1 root root  256 Jan 10 10:00 logs/app.log

# All owned by root! Application runs as user "webapp"
# webapp user can't write logs!

$ sudo -u webapp touch /opt/webapp/logs/test.log
# ❌ Permission denied (root owns it, others only have r-x)
```

**Change ownership recursively:**

```bash
$ sudo chown -R webapp:webapp /opt/webapp/
#              │   │      │
#              │   │      └─ Directory to change (recursively)
#              │   └──────── Group
#              └──────────── Owner
```

**AFTER:**
```bash
$ ls -la /opt/webapp/
drwxr-xr-x 5 webapp webapp 4096 Jan 10 10:00 .
-rwxr-xr-x 1 webapp webapp 2048 Jan 10 10:00 app.py
-rw-r--r-- 1 webapp webapp  512 Jan 10 10:00 config.ini
drwxr-xr-x 2 webapp webapp 4096 Jan 10 10:00 logs/
-rw-r--r-- 1 webapp webapp  256 Jan 10 10:00 logs/app.log

# Now webapp user owns everything!
$ sudo -u webapp touch /opt/webapp/logs/test.log
# ✅ Works! webapp is owner with rwx on logs/ directory
```

**What changed?**
- **Before**: root owned everything → webapp couldn't write
- **After**: webapp owns everything → full control over its own files

---

### 📝 **Example 4: Changing Only the Group**

**Scenario:** Owner stays the same, but you want to change which group has access.

**BEFORE:**
```bash
$ ls -l report.txt
-rw-r--r-- 1 sam engineering 1024 Jan 10 10:00 report.txt
              │   │
              └───└─ Owner: sam, Group: engineering

# Engineering team can read it
```

**Change group to marketing:**

```bash
$ sudo chown :marketing report.txt
#              │
#              └─ Colon with no user = change group only
```

**AFTER:**
```bash
$ ls -l report.txt
-rw-r--r-- 1 sam marketing 1024 Jan 10 10:00 report.txt
              │   │
              └───└─ Owner: sam (unchanged), Group: marketing (changed)

# Now marketing team can read it
# Engineering team lost access (now in "others" category)
```

---

### 🔧 **The chgrp Command (Change Group Only)**

Alternative way to change only the group:

```bash
$ chgrp marketing report.txt
#       │         │
#       │         └─ File
#       └─────────── New group

# Equivalent to: chown :marketing report.txt
```

---

### ⚠️ **Common Ownership Mistakes**

#### **Mistake 1: Forgetting sudo**

```bash
$ chown webapp:webapp /opt/webapp/app.py
# ❌ Operation not permitted

# Fix: You need sudo to change ownership of files you don't own
$ sudo chown webapp:webapp /opt/webapp/app.py
# ✅ Works
```

#### **Mistake 2: Wrong syntax for group**

```bash
# WRONG: Using space
$ sudo chown sam developers file.txt
# Tries to change ownership of TWO files: "developers" and "file.txt"

# CORRECT: Using colon
$ sudo chown sam:developers file.txt
```

#### **Mistake 3: Changing ownership without checking permissions**

```bash
# You change owner to webapp
$ sudo chown webapp:webapp app.py
$ ls -l app.py
-rw------- 1 webapp webapp 2048 Jan 10 10:00 app.py

# But permissions are 600 (rw-------) = only owner can access
# Your nginx runs as www-data, can't read it!

# Fix: Also fix permissions
$ sudo chmod 644 app.py  # Now others can read
```

---

### 📊 **Ownership Change Summary**

| Command | What It Does | Example |
|---------|--------------|---------|
| `chown user file` | Change owner | `chown sam app.py` |
| `chown user:group file` | Change owner + group | `chown sam:developers app.py` |
| `chown :group file` | Change group only | `chown :developers app.py` |
| `chown -R user:group dir/` | Recursive | `chown -R webapp:webapp /opt/` |
| `chgrp group file` | Change group (alternative) | `chgrp developers app.py` |

---

## 🔒 **SPECIAL PERMISSIONS (COMPREHENSIVE GUIDE)**

Beyond the basic rwx permissions, Linux has **3 special permissions** that change HOW programs run and HOW directories behave.

---

### 🎯 **1. SETUID (Set User ID) - 4xxx**

**What it does:** When you execute a file, it runs with the **file owner's permissions**, not YOUR permissions.

---

#### **Why It Exists: The Real-World Problem**

**Scenario:** You want to change your password.

```bash
$ passwd
# This command needs to edit /etc/shadow (contains all passwords)

$ ls -l /etc/shadow
-rw-r----- 1 root shadow 1234 Jan 10 10:00 /etc/shadow
#             │
#             └─ Only root can write to this file
```

**The problem:**
- Regular users need to change their passwords
- But `/etc/shadow` is owned by root, regular users can't write to it
- We can't give everyone write access to `/etc/shadow` (security disaster!)

**The solution: setuid**

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 63736 Jan 10 10:00 /usr/bin/passwd
   │
   └─ 's' instead of 'x' = setuid is enabled
```

**How it works:**
```
1. You (sam) run: passwd
2. Normally: passwd runs as "sam" → can't write to /etc/shadow
3. With setuid: passwd runs as "root" (file owner) → CAN write to /etc/shadow
4. After passwd finishes: you're back to being "sam"
```

---

#### **Setuid Example: Before and After**

**Create a test script:**

```bash
$ cat > checkuser.sh << 'EOF'
#!/bin/bash
echo "Running as user: $(whoami)"
echo "Real user: $USER"
EOF

$ chmod +x checkuser.sh
```

**BEFORE setuid:**

```bash
$ ls -l checkuser.sh
-rwxr-xr-x 1 root root 78 Jan 10 10:00 checkuser.sh
   │
   └─ Normal execute permission

$ ./checkuser.sh
Running as user: sam
Real user: sam

# Script runs as YOU (sam)
```

**Apply setuid:**

```bash
$ sudo chmod u+s checkuser.sh
#              │ │
#              │ └─ Add setuid
#              └─── To user (owner)

# Or numerically:
$ sudo chmod 4755 checkuser.sh
#              │
#              └─ 4 = setuid bit
```

**AFTER setuid:**

```bash
$ ls -l checkuser.sh
-rwsr-xr-x 1 root root 78 Jan 10 10:00 checkuser.sh
   │
   └─ 's' instead of 'x' = setuid enabled

$ ./checkuser.sh
Running as user: root
Real user: sam

# Script runs as root (file owner), even though you're sam!
```

**What changed?**
- **Before**: Script runs with your (sam) permissions
- **After**: Script runs with owner's (root) permissions

---

#### **Setuid Security Warning**

```bash
# ⚠️ NEVER do this on scripts you don't fully trust
# A setuid root script = giving root access!

# Example of danger:
$ cat > dangerous.sh << 'EOF'
#!/bin/bash
# This would let anyone become root!
/bin/bash
EOF

$ sudo chown root:root dangerous.sh
$ sudo chmod 4755 dangerous.sh

$ ./dangerous.sh
# Now you have a root shell! VERY DANGEROUS!
```

**Safe usage:**
- Only use setuid on trusted binaries (like `/usr/bin/passwd`)
- Never use on shell scripts (security risk)
- Carefully audit any setuid programs

---

### 🎯 **2. SETGID (Set Group ID) - 2xxx**

**Two different behaviors:**
1. **On files**: Run with file's group permissions (like setuid but for groups)
2. **On directories**: New files inherit directory's group (THIS IS USEFUL!)

---

#### **Setgid on Files (Less Common)**

Similar to setuid, but uses group instead of owner.

```bash
$ chmod g+s file
# Or numerically: chmod 2755 file

$ ls -l file
-rwxr-sr-x 1 sam developers 1024 Jan 10 10:00 file
      │
      └─ 's' in group position = setgid
```

---

#### **Setgid on Directories (VERY USEFUL!)**

**The Problem:** Shared team directory.

**BEFORE setgid:**

```bash
# Create shared project directory
$ sudo mkdir /opt/team-project
$ sudo chown sam:developers /opt/team-project
$ sudo chmod 775 /opt/team-project

$ ls -ld /opt/team-project
drwxrwxr-x 2 sam developers 4096 Jan 10 10:00 /opt/team-project
                  │
                  └─ Group: developers

# Sam creates a file
$ touch /opt/team-project/sam-file.txt
$ ls -l /opt/team-project/sam-file.txt
-rw-r--r-- 1 sam sam 0 Jan 10 10:00 sam-file.txt
                    │
                    └─ Group is 'sam' (sam's primary group), NOT 'developers'!

# Alice (in developers group) tries to edit it
$ sudo -u alice echo "test" >> /opt/team-project/sam-file.txt
# ❌ Permission denied! (alice is in "others" category, only has r--)
```

**The problem:** Files inherit creator's **primary group**, not directory's group!

---

**Apply setgid to directory:**

```bash
$ sudo chmod g+s /opt/team-project
#              │ │
#              │ └─ setgid
#              └─── On group

# Or numerically:
$ sudo chmod 2775 /opt/team-project
#              │
#              └─ 2 = setgid bit
```

**AFTER setgid:**

```bash
$ ls -ld /opt/team-project
drwxrwsr-x 2 sam developers 4096 Jan 10 10:00 /opt/team-project
      │
      └─ 's' in group position = setgid enabled

# Sam creates a NEW file
$ touch /opt/team-project/new-file.txt
$ ls -l /opt/team-project/new-file.txt
-rw-r--r-- 1 sam developers 0 Jan 10 10:00 new-file.txt
                    │
                    └─ Group is 'developers' (inherited from directory)!

# Alice can now access it!
# But wait, she still can't write (permissions are rw-r--r--)
# We need to fix the default permissions too:

$ chmod 664 /opt/team-project/new-file.txt
$ ls -l /opt/team-project/new-file.txt
-rw-rw-r-- 1 sam developers 0 Jan 10 10:00 new-file.txt
   ││
   │└─ Group can write!
   └── Owner can write

# Now Alice can edit it!
$ sudo -u alice echo "Alice was here" >> /opt/team-project/new-file.txt
# ✅ Works! She's in developers group with rw- permission
```

**What changed?**
- **Before setgid**: New files got creator's primary group (sam) → team couldn't access
- **After setgid**: New files inherit directory's group (developers) → team can access

---

#### **Setgid + umask for Perfect Team Directory**

```bash
# Setup perfect shared directory
$ sudo mkdir /opt/shared
$ sudo chown :developers /opt/shared
$ sudo chmod 2775 /opt/shared
#              ││││
#              │└└└─ rwxrwxr-x (owner+group full, others read/enter)
#              └──── 2 = setgid (inherit group)

# Set umask so new files are group-writable
$ umask 002
#         │
#         └─ New files: 664 (rw-rw-r--), dirs: 775 (rwxrwxr-x)

# Now any file created is automatically group-writable!
$ touch /opt/shared/team-doc.txt
$ ls -l /opt/shared/team-doc.txt
-rw-rw-r-- 1 sam developers 0 Jan 10 10:00 team-doc.txt

# Perfect for team collaboration!
```

---

### 🎯 **3. STICKY BIT - 1xxx**

**What it does:** On a directory, only the **file owner** can delete their files (even if others have write permission on directory).

---

#### **Why It Exists: The /tmp Problem**

**Scenario:** `/tmp` is world-writable (everyone can create files).

**WITHOUT sticky bit:**

```bash
$ ls -ld /tmp
drwxrwxrwx 2 root root 4096 Jan 10 10:00 /tmp
   │││
   └└└─ Everyone has write permission

# Sam creates a file
$ touch /tmp/sam-file.txt

# Alice can DELETE Sam's file! (because she has write on /tmp directory)
$ sudo -u alice rm /tmp/sam-file.txt
# ✅ Works! (BAD - Alice deleted Sam's file!)
```

**The problem:** Anyone can delete anyone's files in a shared directory!

---

**Apply sticky bit:**

```bash
$ sudo chmod +t /tmp
#              │
#              └─ Sticky bit

# Or numerically:
$ sudo chmod 1777 /tmp
#              │
#              └─ 1 = sticky bit
```

**WITH sticky bit:**

```bash
$ ls -ld /tmp
drwxrwxrwt 2 root root 4096 Jan 10 10:00 /tmp
       │
       └─ 't' instead of 'x' = sticky bit enabled

# Sam creates a file
$ touch /tmp/sam-file.txt
$ ls -l /tmp/sam-file.txt
-rw-r--r-- 1 sam sam 0 Jan 10 10:00 sam-file.txt

# Alice tries to delete Sam's file
$ sudo -u alice rm /tmp/sam-file.txt
# ❌ Permission denied! (sticky bit protects it)

# Only Sam (owner) or root can delete it
$ rm /tmp/sam-file.txt
# ✅ Works! (Sam is the owner)
```

**What changed?**
- **Before sticky bit**: Anyone with write on directory can delete ANY file
- **After sticky bit**: Only file owner (or root) can delete their own files

---

### 📊 **Special Permissions Summary**

| Permission | Numeric | Symbol | On Files | On Directories |
|------------|---------|--------|----------|----------------|
| **setuid** | 4xxx | `s` (user) | Runs as file owner | No effect |
| **setgid** | 2xxx | `s` (group) | Runs as file group | New files inherit group |
| **sticky bit** | 1xxx | `t` (others) | No effect | Only owner can delete files |

---

### 🎨 **How Special Permissions Display**

```bash
# Normal permissions
-rwxr-xr-x

# With setuid (owner)
-rwsr-xr-x  # 's' replaces 'x' in owner position
   │
   └─ setuid

# With setgid (group)  
-rwxr-sr-x  # 's' replaces 'x' in group position
      │
      └─ setgid

# With sticky bit (others)
drwxrwxrwt  # 't' replaces 'x' in others position
        │
        └─ sticky bit

# All together (rare!)
-rwsrwsrwt  # setuid + setgid + sticky bit
   │  │  │
   │  │  └─ sticky bit
   │  └──── setgid
   └─────── setuid
```

---

### 🔢 **Setting Special Permissions**

```bash
# Symbolic method
chmod u+s file    # Add setuid
chmod g+s dir     # Add setgid
chmod +t dir      # Add sticky bit
chmod u-s file    # Remove setuid

# Numeric method (4-digit)
chmod 4755 file   # setuid + rwxr-xr-x
chmod 2775 dir    # setgid + rwxrwxr-x
chmod 1777 dir    # sticky + rwxrwxrwx
chmod 6755 file   # setuid + setgid + rwxr-xr-x
chmod 7777 dir    # all special bits + rwxrwxrwx (DON'T DO THIS!)

# 4-digit breakdown
# [special][owner][group][others]
#    │       │      │      │
#    │       │      │      └─ others (0-7)
#    │       │      └──────── group (0-7)
#    │       └─────────────── owner (0-7)
#    └─────────────────────── 4=setuid, 2=setgid, 1=sticky, or add them
```

---

### ⚠️ **When to Use Special Permissions**

| Use Case | Permission | Command |
|----------|------------|---------|
| Shared team directory | setgid | `chmod 2775 /opt/team-project` |
| Public upload directory | sticky bit | `chmod 1777 /var/uploads` |
| System binaries (careful!) | setuid | Usually already set, don't touch |
| Normal files/dirs | NONE | Use regular permissions (755, 644, etc.) |

**General rule:** Don't use special permissions unless you have a specific reason!

---

**Symbolic Method** (easier to remember):
```bash
chmod u+x script.sh                   # Add execute for owner
chmod g+w file.txt                    # Add write for group
chmod o-r file.txt                    # Remove read from others
chmod a+r file.txt                    # Add read for all (owner, group, others)
chmod u=rwx,g=rx,o=r file.txt        # Set exact permissions

# u = user (owner), g = group, o = others, a = all
# + = add, - = remove, = = set exactly
```

**Numeric Method** (faster):
```bash
chmod 755 script.sh                   # rwxr-xr-x
chmod 644 config.txt                  # rw-r--r--
chmod 600 ~/.ssh/id_rsa               # rw------- (SSH key)
chmod 700 ~/.ssh/                     # rwx------ (SSH directory)
```

**Recursive** (entire directory tree):
```bash
chmod -R 755 /var/www/html/           # All files and subdirs
```

**Production Examples**:
```bash
# Make deployment script executable
chmod +x /opt/myapp/deploy.sh

# Secure SSH keys
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 700 ~/.ssh/

# Fix web directory permissions
sudo chmod -R 755 /var/www/html/
sudo chmod -R 644 /var/www/html/*.html

# Application directory
chmod 755 /opt/myapp/bin/*            # Binaries executable
chmod 644 /opt/myapp/config/*         # Configs readable
chmod 600 /opt/myapp/config/secrets.yml  # Secrets private
```

---

## �️ **CHANGING PERMISSIONS (chmod Command)**

Now that you understand ownership, let's learn how to change the actual permissions (rwx).

### **chmod** - Change Mode (Permissions)

`umask` sets default permissions for new files/directories.

```bash
umask                                 # Show current umask (e.g., 0022)
umask 0027                           # Set new umask

# Calculation
# Default for files: 666 (rw-rw-rw-)
# Default for dirs:  777 (rwxrwxrwx)
# Subtract umask:    022 (----w--w-)
# Result:            644 for files, 755 for dirs
```

**Production Setting**:
```bash
# In ~/.bashrc or /etc/profile
umask 0027                            # Files: 640, Dirs: 750 (stricter)
umask 0022                            # Files: 644, Dirs: 755 (common default)
```

---

## ⚠️ Common Permission Mistakes

### **Mistake 1: Using 777**

```bash
# WRONG: World-writable = security hole
chmod 777 /opt/myapp/

# CORRECT: Only give needed permissions
chmod 755 /opt/myapp/
```

### **Mistake 2: Running Everything as Root**

```bash
# WRONG: Running app as root
sudo ./app

# CORRECT: Create dedicated user
sudo useradd -r -s /bin/false appuser
sudo chown -R appuser:appuser /opt/myapp
su - appuser -c "/opt/myapp/start.sh"
```

### **Mistake 3: Forgetting SSH Key Permissions**

```bash
# If SSH key has wrong permissions, SSH will refuse to use it
chmod 600 ~/.ssh/id_rsa               # Private key
chmod 644 ~/.ssh/id_rsa.pub           # Public key
chmod 700 ~/.ssh/                     # SSH directory
```

### **Mistake 4: Recursive chmod on /**

```bash
# CATASTROPHIC: Don't do this
sudo chmod -R 777 /

# If you accidentally ran something bad, restore from backup
```

---

## 🛠️ Production Debugging Scenarios

### **Scenario 1: "Permission denied" when running script**

```bash
$ ./deploy.sh
-bash: ./deploy.sh: Permission denied

# Diagnose
ls -l deploy.sh
# -rw-r--r-- (no execute permission)

# Fix
chmod +x deploy.sh
```

### **Scenario 2: Web server can't read files**

```bash
# Check ownership
ls -l /var/www/html/
# -rw-r--r-- root root index.html

# Check what user nginx runs as
ps aux | grep nginx
# nginx: worker process is nobody

# Fix: change ownership
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/
```

### **Scenario 3: Application can't write logs**

```bash
# Check directory permissions
ls -ld /var/log/myapp/
# drwxr-xr-x root root

# Fix: make it writable by app user
sudo chown -R appuser:appuser /var/log/myapp/
sudo chmod 755 /var/log/myapp/
```

### **Scenario 4: SSH key not working**

```bash
# SSH requires strict permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 644 ~/.ssh/authorized_keys
```

---

## 🧪 **MINI-TASK: Secure File Management**

**Objective**: Practice file operations and implement proper security permissions.

**Scenario**: You're setting up a simple application directory structure with proper security.

**Tasks**:

1. Create the following directory structure:
   ```
   /home/sam/app-deployment/
   ├── bin/           (executables)
   ├── config/        (configuration files)
   ├── logs/          (log files)
   └── data/          (application data)
   ```

2. Create the following files with **proper permissions**:
   - `/home/sam/app-deployment/bin/startup.sh` (executable script)
   - `/home/sam/app-deployment/config/app.conf` (readable by owner only)
   - `/home/sam/app-deployment/config/database.yml` (readable/writable by owner only - contains secrets)
   - `/home/sam/app-deployment/logs/app.log` (writable by owner)

3. Write to `startup.sh`:
   ```bash
   #!/bin/bash
   echo "Application starting..."
   date >> /home/sam/app-deployment/logs/app.log
   ```

4. Set these **exact permissions**:
   - `bin/startup.sh`: 750 (rwxr-x---)
   - `config/app.conf`: 644 (rw-r--r--)
   - `config/database.yml`: 600 (rw-------)
   - `logs/app.log`: 644 (rw-r--r--)
   - `logs/` directory: 755 (rwxr-xr-x)

5. Test the script:
   - Run `./startup.sh` from the bin directory
   - Verify it writes to the log file

6. Create a report: `/home/sam/DevopsRoadmap/Phase-1-Linux/permissions-report.txt` containing:
   - Complete `ls -la` output of the entire `/home/sam/app-deployment/` structure
   - Explanation of why you chose each permission setting
   - The commands you used to create and set permissions

**Security Challenge**: Explain what would happen if:
- `startup.sh` had 777 permissions?
- `database.yml` had 644 permissions?
- `/logs` directory had 777 permissions?

**Expected Time**: 45-60 minutes

**Deliverable**: Show me your `permissions-report.txt` and demonstrate the directory structure.

---

## 📌 Quick Reference Card

```bash
# File Operations
touch file                  # Create empty file
mkdir -p path/to/dir        # Create directories
cp -r source dest           # Copy recursively
rm -rf dir/                 # Delete (DANGEROUS)

# Viewing
cat file                    # Display all
head -n 20 file             # First 20 lines
tail -f file                # Follow live
less file                   # Paginated view

# Permissions
ls -l file                  # View permissions
chmod 755 file              # Change permissions (rwxr-xr-x)
chmod u+x file              # Add execute for owner
chown user:group file       # Change owner
chown -R user:group dir/    # Recursive

# Common Patterns
chmod 644 config            # Config files
chmod 600 secrets           # Secret files
chmod 755 scripts           # Executable scripts
chmod 700 ~/.ssh/           # SSH directory
```

---

## 📚 MY LEARNING LOG

### ✅ **Completion Date:** January 11, 2026

---

### 🤔 **Questions & Doubts I Had:**

#### **DOUBT 1: Hard Links and Link Count Confusion**

**My Question:**
```bash
satish@sam-VMware-Virtual-Platform:~$ ls -ld
drwxr-x--- 3 satish satish 4096 Jan 11 12:35 .
satish@sam-VMware-Virtual-Platform:~$ ls -l
total 4
drwxrwxr-x 2 satish satish 4096 Jan 11 12:35 satish
```

"Why is it showing hard link as 3 and 2 for different ls command for the same file?
What are the hard links here?"

---

**INSTRUCTOR CLARIFICATION:**

Let me explain what's happening here step by step.

**First, understand what you're looking at:**

```bash
ls -ld     # Shows info about the DIRECTORY ITSELF (~)
ls -l      # Shows info about CONTENTS inside the directory
```

**Let's break down the output:**

```bash
$ ls -ld
drwxr-x--- 3 satish satish 4096 Jan 11 12:35 .
           │
           └─ Hard link count = 3
```

This `3` is the hard link count for the home directory `/home/satish` itself.

```bash
$ ls -l
drwxrwxr-x 2 satish satish 4096 Jan 11 12:35 satish
           │
           └─ Hard link count = 2
```

This `2` is the hard link count for the subdirectory `/home/satish/satish`.

They're **different directories**, so they have **different link counts**!

---

**Understanding Directory Hard Link Count:**

Every directory has a hard link count that follows this rule:

```
Link Count = 2 + (number of subdirectories inside it)
```

**Why 2?**
1. The directory name itself (e.g., `satish`)
2. The `.` entry inside the directory (self-reference)

**Additional +1 for each subdirectory:**
Each subdirectory adds `+1` because it contains `..` (parent reference).

---

**Let's Trace Your Example:**

**Starting state:**
```bash
# Create user satish
$ sudo useradd -m satish
# Home directory created: /home/satish/
```

**After user creation:**
```bash
$ sudo ls -ld /home/satish/
drwxr-x--- 2 satish satish 4096 Jan 11 12:35 /home/satish/
           │
           └─ Link count = 2

# Why 2?
# 1. /home/satish (the directory name)
# 2. /home/satish/. (self-reference)
# No subdirectories yet, so just 2
```

**After creating subdirectory:**
```bash
$ sudo -u satish mkdir /home/satish/satish
# Now a subdirectory exists inside /home/satish/
```

**Now check again:**
```bash
$ sudo ls -ld /home/satish/
drwxr-x--- 3 satish satish 4096 Jan 11 12:35 /home/satish/
           │
           └─ Link count = 3

# Why 3?
# 1. /home/satish (the directory name)
# 2. /home/satish/. (self-reference)
# 3. /home/satish/satish/.. (subdirectory's parent reference)
```

**The subdirectory itself:**
```bash
$ sudo ls -ld /home/satish/satish/
drwxrwxr-x 2 satish satish 4096 Jan 11 12:35 /home/satish/satish/
           │
           └─ Link count = 2

# Why 2?
# 1. /home/satish/satish (the directory name)
# 2. /home/satish/satish/. (self-reference)
# No subdirectories inside it yet, so just 2
```

---

**Visual Representation:**

```
/home/satish/                    ← Link count = 3
├── .                            ← Reference #1 (self)
├── ..                           ← Points to /home/
└── satish/                      ← This adds +1 to parent
    ├── .                        ← Reference #1 for subdirectory
    └── ..                       ← Reference #3 back to parent (/home/satish/)

So /home/satish/ link count:
1. /home/satish (name itself)
2. /home/satish/. (dot reference)
3. /home/satish/satish/.. (subdirectory's parent reference)
= 3 total

And /home/satish/satish/ link count:
1. /home/satish/satish (name itself)
2. /home/satish/satish/. (dot reference)
= 2 total
```

---

**Test This Yourself:**

```bash
# Create a test directory
mkdir ~/testdir
ls -ld ~/testdir
# drwxrwxr-x 2 sam sam 4096 Jan 11 15:00 /home/sam/testdir
#            └─ Link count = 2

# Add one subdirectory
mkdir ~/testdir/sub1
ls -ld ~/testdir
# drwxrwxr-x 3 sam sam 4096 Jan 11 15:01 /home/sam/testdir
#            └─ Link count = 3 (added 1)

# Add another subdirectory
mkdir ~/testdir/sub2
ls -ld ~/testdir
# drwxrwxr-x 4 sam sam 4096 Jan 11 15:02 /home/sam/testdir
#            └─ Link count = 4 (added another 1)

# Formula confirmed: 2 + number_of_subdirectories
# 2 + 2 = 4 ✓
```

---

**KEY TAKEAWAY:**
- **Files**: Hard link count = number of names pointing to same data
- **Directories**: Hard link count = 2 + (number of subdirectories)
- `.` = self-reference (adds 1)
- `..` = parent reference (doesn't count for itself, counts for parent)

---

#### **DOUBT 2: Setuid and File Modification Concerns**

**My Question:**
```
-rwsr-xr-x 1 root root 64152 May 30  2024 /usr/bin/passwd

"Now here if any user can write in this file with root permission, because of s bit is set,
wouldn't that user be able to see and modify all the content of the file?"
```

---

**INSTRUCTOR CLARIFICATION:**

**Great security question!** But you're confusing two different things:

1. **Permissions on the FILE itself** (who can read/write/execute the file)
2. **Permissions the FILE RUNS WITH** (setuid - what privileges it gets when executed)

Let me clarify:

---

**Understanding the Permissions:**

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 64152 May 30  2024 /usr/bin/passwd
 │││  │ │
 │││  │ └─ Others can execute (x)
 │││  └─── Group can execute (x)
 ││└────── Owner can execute, WITH setuid (s)
 │└─────── Owner can write (w) - ONLY ROOT
 └──────── Owner can read (r) - ONLY ROOT
```

**Breaking it down:**

```
Position:  Owner    Group    Others
           rwx      r-x      r-x
           ││└─s    │││      │││
           │└──w    │└─x     │└─x
           └───r    └──r     └──r
```

**Who can MODIFY the file?**
- **Owner (root)**: Has `w` (write) permission ✅
- **Group (root)**: NO write permission ❌
- **Others (you)**: NO write permission ❌

**Can a regular user modify /usr/bin/passwd file?**
**NO!** You don't have write permission on the file itself.

```bash
# Try to modify it as regular user
$ echo "malicious code" >> /usr/bin/passwd
bash: /usr/bin/passwd: Permission denied
```

**Why? Look at the permissions again:**
```
-rwsr-xr-x
 │└─ Only owner (root) can write
 └── You're not root, so you can't write
```

---

**So What Does Setuid Actually Do?**

Setuid ONLY affects **WHEN YOU RUN** the program, not when you try to modify it.

**WITHOUT setuid:**
```bash
# Normal executable
-rwxr-xr-x 1 root root 1234 Jan 11 10:00 /usr/bin/normalprogram

# When you run it:
$ /usr/bin/normalprogram
# It runs as YOU (sam), with YOUR permissions
```

**WITH setuid:**
```bash
# Setuid executable
-rwsr-xr-x 1 root root 64152 May 30  2024 /usr/bin/passwd

# When you run it:
$ /usr/bin/passwd
# It runs as ROOT (owner), with ROOT permissions
# But ONLY while the program is running
# After it exits, you're back to being sam
```

---

**Real-World Example: passwd Command**

**The problem it solves:**
```bash
# Passwords stored in /etc/shadow
$ ls -l /etc/shadow
-rw-r----- 1 root shadow 1234 Jan 11 10:00 /etc/shadow
   │
   └─ Only root can write

# You need to change your password, but you're not root!
# How can you modify /etc/shadow?
```

**The solution: setuid on /usr/bin/passwd**

```bash
# When you run passwd:
$ passwd

# What happens (simplified):
1. You (sam) execute /usr/bin/passwd
2. Because of setuid, passwd runs as ROOT
3. passwd (running as root) can write to /etc/shadow
4. It changes YOUR password (it checks who you are)
5. passwd exits
6. You're back to being sam (not root anymore)
```

**Security measures in passwd:**
- It's a **compiled binary** (C code), not a shell script
- It has **built-in security checks** (you can only change YOUR password, not others')
- The source code is **audited** by security experts
- Even though it runs as root, it's **programmed** to only modify specific things

---

**Why Can't You Modify the File?**

```bash
# Let's verify
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 64152 May 30  2024 /usr/bin/passwd
   │
   └─ You don't have 'w' (write) permission

# Try to modify
$ echo "hacked" >> /usr/bin/passwd
bash: /usr/bin/passwd: Permission denied

# Try to replace it
$ cp /bin/ls /usr/bin/passwd
cp: cannot create regular file '/usr/bin/passwd': Permission denied

# Even with setuid, you CANNOT modify the file itself
```

**To modify it, you'd need to:**
1. Become root: `sudo vim /usr/bin/passwd` (requires sudo privilege)
2. Or actually BE the root user

---

**Summary Table:**

| Action | Regular User | With Setuid | Explanation |
|--------|--------------|-------------|-------------|
| **Read file** | ✅ Yes (r--) | ✅ Yes | Everyone has read permission |
| **Write/Modify file** | ❌ No | ❌ No | Only owner (root) can write |
| **Execute file** | ✅ Yes (--x) | ✅ Yes | Everyone can execute |
| **Run with owner's privileges** | ❌ No | ✅ Yes | Setuid makes it run as owner |

**Key Point:** Setuid affects RUNTIME privileges (when program runs), NOT file modification privileges.

---

#### **DOUBT 3: Setuid on Shell Scripts**

**My Observation:**
"You can't get desired output by running the shell script to prove s bit is set, as Linux ignores it for shell scripts but works for binary compiled."

---

**INSTRUCTOR CLARIFICATION:**

✅ **You're ABSOLUTELY CORRECT!** This is an important security feature.

**Why Linux Ignores Setuid on Scripts:**

**Security reasons:**
1. **Shell scripts are text files** - easy to tamper with
2. **Shell interpreters are complex** - many ways to exploit
3. **Environment variables can be exploited** - PATH manipulation
4. **Race conditions** - attacker could modify script between permission check and execution

---

**Example of the Danger (if setuid worked on scripts):**

```bash
# Imagine if this worked (it doesn't):
$ cat /usr/local/bin/backup.sh
#!/bin/bash
# Setuid script owned by root
cp /etc/important-config /backup/

$ ls -l /usr/local/bin/backup.sh
-rwsr-xr-x 1 root root 100 Jan 11 10:00 backup.sh

# Attacker could exploit this:
# 1. Manipulate PATH variable
export PATH=/tmp/evil:$PATH

# 2. Create fake 'cp' command
cat > /tmp/evil/cp << 'EOF'
#!/bin/bash
# Fake cp that gives attacker root shell
/bin/bash
EOF
chmod +x /tmp/evil/cp

# 3. Run the script
./backup.sh
# Now attacker has root shell! (if setuid worked)
```

**This is why Linux kernel IGNORES setuid on scripts.**

---

**How to Test This:**

```bash
# Create a shell script
$ cat > test.sh << 'EOF'
#!/bin/bash
echo "Running as: $(whoami)"
echo "Effective UID: $(id -u)"
EOF

$ chmod 755 test.sh

# Make it owned by root
$ sudo chown root:root test.sh

# Set setuid bit
$ sudo chmod u+s test.sh

# Check permissions
$ ls -l test.sh
-rwsr-xr-x 1 root root 80 Jan 11 15:00 test.sh
   │
   └─ Setuid bit is SET

# Run it
$ ./test.sh
Running as: sam
Effective UID: 1000

# ❌ Still runs as sam, NOT as root!
# Linux kernel ignores setuid on shell scripts
```

---

**But Setuid DOES Work on Compiled Binaries:**

```bash
# Check passwd (compiled C program)
$ file /usr/bin/passwd
/usr/bin/passwd: ELF 64-bit LSB pie executable
                 └─ It's a COMPILED BINARY

$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 64152 May 30  2024 /usr/bin/passwd

# Run it
$ /usr/bin/passwd
# ✅ Runs as root (works because it's compiled)
```

---

**Which Script Interpreters Support Setuid?**

| Interpreter | Setuid Works? | Why |
|-------------|---------------|-----|
| Shell scripts (`#!/bin/bash`, `#!/bin/sh`) | ❌ No | Security risk, kernel ignores |
| Perl scripts (`#!/usr/bin/perl`) | ⚠️ Maybe | Only if perl compiled with setuid support |
| Python scripts (`#!/usr/bin/python`) | ❌ No | Security risk, ignored |
| Compiled C/C++ binaries | ✅ Yes | Safe, carefully coded |

---

**Production Reality:**

If you need setuid functionality with a script, you must:

1. **Write a C wrapper** (compiled binary with setuid)
2. The wrapper calls your script (without setuid)

**Example:**

```c
// wrapper.c
#include <unistd.h>
#include <stdlib.h>

int main() {
    // Set real UID to effective UID (become root)
    setuid(0);
    
    // Execute the actual script
    system("/usr/local/bin/actual-script.sh");
    
    return 0;
}

// Compile:
// gcc -o wrapper wrapper.c
// sudo chown root:root wrapper
// sudo chmod 4755 wrapper
```

Now `wrapper` (compiled binary) has setuid and calls your script.

**But even this is dangerous!** Better solutions:
- Use `sudo` with specific command allowances
- Use systemd services running as specific users
- Use capabilities instead of full root

---

**Key Takeaways:**

1. ✅ Setuid on compiled binaries = Works and is used (e.g., passwd)
2. ❌ Setuid on shell scripts = Ignored by Linux kernel (security)
3. ⚠️ If you need setuid functionality, use sudo or compiled wrapper
4. 🔒 Setuid is powerful and dangerous - use sparingly

**You demonstrated excellent security awareness by questioning this!**

---

### 💪 **Skills Acquired:**

✅ Understand Linux permission model (rwx, owner/group/others)  
✅ Calculate and set numeric permissions (755, 644, 600)  
✅ Secure sensitive files (600 for secrets)  
✅ Create and manage proper directory structures  
✅ Write and secure shell scripts  
✅ Understand security implications of bad permissions  
✅ Grasp hard link concepts for directories  
✅ Understand setuid security (what it does and doesn't do)  
✅ Know why setuid doesn't work on scripts  

---

**Status: MODULE 2 COMPLETED ✅**

**Grade: 90/100** - Excellent work with strong security awareness!

**Ready for Module 3: Process Management**

---

**Complete this mini-task, then we'll move to Module 3: Process Management.**
