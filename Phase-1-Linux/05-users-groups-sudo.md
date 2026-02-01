# Module 5: Users, Groups & sudo

## 🎯 Why This Matters in Production

**Scenario**: A new developer joins your team. They need:
- SSH access to the server
- Ability to deploy code to `/var/www/app`
- Permission to restart services
- But NOT access to database credentials or other users' files

You need to instantly know:
- **How do I create a user account?**
- **How do I grant specific permissions?**
- **How do I allow them to run certain commands as root?**
- **How do I audit who did what?**

**In production, user management = access control = security.**

---

## 🧠 **USERS AND GROUPS FUNDAMENTALS**

### **The Unix Security Model**

Linux is a **multi-user operating system**. Every file, process, and resource has an owner.

```
┌─────────────────────────────────────────────────────────────┐
│                    LINUX SECURITY MODEL                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   USER (UID)          GROUP (GID)         OTHERS            │
│   ┌─────────┐         ┌─────────┐         ┌─────────┐       │
│   │  sam    │         │  dev    │         │ everyone│       │
│   │UID: 1000│         │GID: 1001│         │  else   │       │
│   └─────────┘         └─────────┘         └─────────┘       │
│        │                   │                   │             │
│        └───────────────────┴───────────────────┘             │
│                            │                                 │
│                    FILE PERMISSIONS                          │
│                    rwx  rwx  rwx                             │
│                    user grp  other                           │
└─────────────────────────────────────────────────────────────┘
```

---

### **Users: The Basics**

Every user has:

| Attribute | Description | Example |
|-----------|-------------|---------|
| **Username** | Human-readable name | `sam`, `www-data` |
| **UID** | Unique numeric ID | `1000`, `33` |
| **Primary Group** | Default group for new files | `sam`, `www-data` |
| **Home Directory** | Personal workspace | `/home/sam` |
| **Login Shell** | Command interpreter | `/bin/bash` |
| **Password** | Authentication (hashed) | Stored in `/etc/shadow` |

---

### **Types of Users**

```
┌─────────────────────────────────────────────────────────────┐
│                      USER TYPES                              │
├──────────────┬──────────────┬───────────────────────────────┤
│    ROOT      │   SYSTEM     │          REGULAR              │
│   UID = 0    │  UID 1-999   │        UID 1000+              │
├──────────────┼──────────────┼───────────────────────────────┤
│ Superuser    │ Service accts│ Human users                   │
│ ALL powers   │ No login     │ Interactive login             │
│ Can do       │ Run daemons  │ Home directory                │
│ ANYTHING     │ nginx, mysql │ sam, alice, bob               │
└──────────────┴──────────────┴───────────────────────────────┘
```

**UID Ranges (Debian/Ubuntu):**
- `0` — root (superuser)
- `1-999` — system/service accounts
- `1000+` — regular human users

---

### **Where User Info Is Stored**

#### **/etc/passwd — User Database**

```bash
$ cat /etc/passwd | head -5
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
sam:x:1000:1000:Sam Smith:/home/sam:/bin/bash
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
postgres:x:109:117:PostgreSQL administrator:/var/lib/postgresql:/bin/bash
```

**Field breakdown:**
```
sam:x:1000:1000:Sam Smith:/home/sam:/bin/bash
 │  │  │    │      │          │        │
 │  │  │    │      │          │        └─ Login shell
 │  │  │    │      │          └─ Home directory
 │  │  │    │      └─ GECOS (full name, contact info)
 │  │  │    └─ Primary GID
 │  │  └─ UID
 │  └─ Password placeholder (actual in /etc/shadow)
 └─ Username
```

#### **/etc/shadow — Password Hashes (root only)**

```bash
$ sudo cat /etc/shadow | grep sam
sam:$6$rounds=5000$salt$hashedpassword:19000:0:99999:7:::
```

**Why shadow?**
- `/etc/passwd` is world-readable (needed for UID→name lookups)
- Password hashes must be protected → separate file, root-only

#### **/etc/group — Group Database**

```bash
$ cat /etc/group | head -5
root:x:0:
sudo:x:27:sam
www-data:x:33:
sam:x:1000:
dev:x:1001:sam,alice,bob
```

**Field breakdown:**
```
dev:x:1001:sam,alice,bob
 │  │  │     │
 │  │  │     └─ Group members (supplementary)
 │  │  └─ GID
 │  └─ Password placeholder (rarely used)
 └─ Group name
```

---

### 🎯 **Running Example: Web Application Team**

Throughout this module, we'll manage this **production team**:

```
┌─────────────────────────────────────────────────────────────┐
│                    COMPANY: TechCorp                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  GROUPS:                                                     │
│  ├─ dev (GID: 1001)      → Developers                       │
│  ├─ ops (GID: 1002)      → Operations/DevOps                │
│  ├─ deploy (GID: 1003)   → Can deploy to /var/www           │
│  └─ dba (GID: 1004)      → Database administrators          │
│                                                              │
│  USERS:                                                      │
│  ├─ sam (UID: 1000)      → DevOps Engineer [ops, deploy]    │
│  ├─ alice (UID: 1001)    → Developer [dev, deploy]          │
│  ├─ bob (UID: 1002)      → Developer [dev]                  │
│  └─ carol (UID: 1003)    → DBA [dba, ops]                   │
│                                                              │
│  SERVICE ACCOUNTS:                                           │
│  ├─ www-data (UID: 33)   → Nginx/web server                 │
│  └─ postgres (UID: 109)  → PostgreSQL                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 👤 **MANAGING USERS**

### **Viewing User Information**

```bash
# Current user
$ whoami
sam

# Current user's UID and groups
$ id
uid=1000(sam) gid=1000(sam) groups=1000(sam),27(sudo),1001(dev)

# Another user's info
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),1001(dev),1003(deploy)

# Who's logged in?
$ who
sam      pts/0        2026-01-31 10:00 (192.168.1.100)
alice    pts/1        2026-01-31 10:30 (192.168.1.101)

# More detail
$ w
 10:45:00 up 2 days,  3:00,  2 users,  load average: 0.15, 0.10, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
sam      pts/0    192.168.1.100    10:00    0.00s  0.05s  0.00s w
alice    pts/1    192.168.1.101    10:30    5:00   0.02s  0.02s vim app.py
```

---

### **Creating Users with useradd**

#### **Basic Syntax**

```bash
useradd [OPTIONS] USERNAME
```

#### **Common Options**

| Option | Purpose | Example |
|--------|---------|---------|
| `-m` | Create home directory | `-m` |
| `-d` | Specify home directory | `-d /home/alice` |
| `-s` | Set login shell | `-s /bin/bash` |
| `-g` | Set primary group | `-g dev` |
| `-G` | Add to supplementary groups | `-G sudo,docker` |
| `-c` | Set comment (full name) | `-c "Alice Smith"` |
| `-u` | Set specific UID | `-u 1500` |
| `-e` | Account expiration date | `-e 2026-12-31` |
| `-r` | Create system account | `-r` (no home, UID < 1000) |

---

#### **Example 1: Create Regular User**

```bash
# Create user alice with home directory and bash shell
$ sudo useradd -m -s /bin/bash -c "Alice Smith" alice

# Verify
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice)

$ ls -la /home/alice
drwxr-x--- 2 alice alice 4096 Jan 31 10:00 .

# Set password
$ sudo passwd alice
New password: 
Retype new password:
passwd: password updated successfully
```

**BEFORE:**
```bash
$ id alice
id: 'alice': no such user
```

**AFTER:**
```bash
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice)

$ grep alice /etc/passwd
alice:x:1001:1001:Alice Smith:/home/alice:/bin/bash
```

---

#### **Example 2: Create Developer with Groups**

```bash
# Create bob as developer with deploy access
$ sudo useradd -m -s /bin/bash -c "Bob Jones" -G dev,deploy bob

$ id bob
uid=1002(bob) gid=1002(bob) groups=1002(bob),1001(dev),1003(deploy)
```

---

#### **Example 3: Create System Account (No Login)**

```bash
# Create service account for an application
$ sudo useradd -r -s /usr/sbin/nologin -c "MyApp Service" myapp

$ id myapp
uid=998(myapp) gid=998(myapp) groups=998(myapp)

$ grep myapp /etc/passwd
myapp:x:998:998:MyApp Service:/home/myapp:/usr/sbin/nologin

# Cannot login
$ sudo su - myapp
This account is currently not available.
```

---

### **The adduser Alternative (Debian/Ubuntu)**

`adduser` is a friendlier wrapper around `useradd`:

```bash
$ sudo adduser carol
Adding user `carol' ...
Adding new group `carol' (1003) ...
Adding new user `carol' (1003) with group `carol' ...
Creating home directory `/home/carol' ...
Copying files from `/etc/skel' ...
New password: 
Retype new password:
passwd: password updated successfully
Changing the user information for carol
Enter the new value, or press ENTER for the default
        Full Name []: Carol Davis
        Room Number []: 
        Work Phone []: 
        Home Phone []: 
        Other []: 
Is the information correct? [Y/n] Y
```

**useradd vs adduser:**

| Feature | useradd | adduser |
|---------|---------|---------|
| Home directory | Need `-m` flag | Automatic |
| Password | Separate `passwd` command | Prompts during creation |
| Interactive | No | Yes |
| Portability | All Linux | Debian/Ubuntu |
| Scripts | Better for automation | Better for manual |

---

### **Modifying Users with usermod**

```bash
usermod [OPTIONS] USERNAME
```

#### **Common Operations**

```bash
# Change shell
$ sudo usermod -s /bin/zsh alice

# Change home directory (and move contents)
$ sudo usermod -d /home/alice_new -m alice

# Add to supplementary group (APPEND!)
$ sudo usermod -aG docker alice
#              └─ CRITICAL: -a means APPEND
#                 Without -a, replaces ALL groups!

# Lock account (disable login)
$ sudo usermod -L alice

# Unlock account
$ sudo usermod -U alice

# Change username
$ sudo usermod -l alice_new alice

# Set account expiration
$ sudo usermod -e 2026-12-31 alice
```

---

#### **⚠️ CRITICAL: The -aG Trap**

```bash
# WRONG - Removes alice from ALL other groups!
$ sudo usermod -G docker alice
# Alice is now ONLY in docker group

# CORRECT - Appends docker to existing groups
$ sudo usermod -aG docker alice
# Alice keeps all groups AND gets docker
```

**BEFORE (wrong command):**
```bash
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),1001(dev),1003(deploy),27(sudo)
```

**AFTER `usermod -G docker alice` (WRONG):**
```bash
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),999(docker)
# Lost dev, deploy, sudo!
```

**AFTER `usermod -aG docker alice` (CORRECT):**
```bash
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),1001(dev),1003(deploy),27(sudo),999(docker)
# All groups preserved + docker added
```

---

### **Deleting Users with userdel**

```bash
# Delete user (keep home directory)
$ sudo userdel bob

# Delete user AND home directory
$ sudo userdel -r bob
#           └─ removes /home/bob and mail spool

# Force delete even if logged in
$ sudo userdel -f bob
```

**Production consideration:**
```bash
# Before deleting, check what they own
$ find / -user bob 2>/dev/null
/home/bob
/var/www/app/uploads/bob_file.txt

# Reassign ownership before deletion
$ sudo chown alice:dev /var/www/app/uploads/bob_file.txt
$ sudo userdel -r bob
```

---

### **Password Management**

```bash
# Set/change password
$ sudo passwd alice
New password:
Retype new password:

# User changes their own password
$ passwd
Changing password for sam.
Current password:
New password:

# Force password change on next login
$ sudo passwd -e alice
passwd: password expiry information changed.

# Lock account (prefix ! to hash in /etc/shadow)
$ sudo passwd -l alice
passwd: password expiry information changed.

# Unlock account
$ sudo passwd -u alice

# Check password status
$ sudo passwd -S alice
alice P 01/31/2026 0 99999 7 -1
#     │     │      │   │   │  │
#     │     │      │   │   │  └─ Inactivity period
#     │     │      │   │   └─ Warning days
#     │     │      │   └─ Maximum days
#     │     │      └─ Minimum days
#     │     └─ Last change date
#     └─ P=usable, L=locked, NP=no password
```

---

### **Password Aging with chage**

```bash
# View password aging info
$ sudo chage -l alice
Last password change                    : Jan 31, 2026
Password expires                        : never
Password inactive                       : never
Account expires                         : never
Minimum number of days between password change  : 0
Maximum number of days between password change  : 99999
Number of days of warning before password expires: 7

# Set password to expire in 90 days
$ sudo chage -M 90 alice

# Set minimum days between changes (prevent rapid changes)
$ sudo chage -m 7 alice

# Force password change on next login
$ sudo chage -d 0 alice

# Set account expiration date
$ sudo chage -E 2026-12-31 alice
```

**Production policy example:**
```bash
# Enforce password policy for new employee
$ sudo chage -m 1 -M 90 -W 14 alice
#           │    │      └─ Warn 14 days before expiry
#           │    └─ Must change every 90 days
#           └─ Can't change more than once per day
```

---

## 👥 **MANAGING GROUPS**

### **Viewing Group Information**

```bash
# Current user's groups
$ groups
sam sudo dev docker

# Another user's groups
$ groups alice
alice : alice dev deploy

# All groups on system
$ cat /etc/group

# Group details
$ getent group dev
dev:x:1001:sam,alice,bob
```

---

### **Creating Groups**

```bash
# Create group
$ sudo groupadd dev

# Create with specific GID
$ sudo groupadd -g 2000 contractors

# Verify
$ getent group dev
dev:x:1001:
```

---

### **Adding Users to Groups**

```bash
# Method 1: usermod (most common)
$ sudo usermod -aG dev alice

# Method 2: gpasswd
$ sudo gpasswd -a alice dev
Adding user alice to group dev

# Add multiple users
$ sudo gpasswd -M alice,bob,carol dev
# WARNING: This REPLACES all members!
```

---

### **Removing Users from Groups**

```bash
# Using gpasswd
$ sudo gpasswd -d bob dev
Removing user bob from group dev

# Verify
$ groups bob
bob : bob
```

---

### **Modifying and Deleting Groups**

```bash
# Rename group
$ sudo groupmod -n developers dev

# Change GID
$ sudo groupmod -g 2001 developers

# Delete group (must have no primary members)
$ sudo groupdel contractors
```

---

### **Real-World Example: Project Team Setup**

**Scenario:** Set up groups for a new project team.

```bash
# Step 1: Create groups
$ sudo groupadd dev
$ sudo groupadd ops
$ sudo groupadd deploy

# Step 2: Create shared directory
$ sudo mkdir -p /var/www/project
$ sudo chown root:deploy /var/www/project
$ sudo chmod 2775 /var/www/project
#           │││└─ others: r-x
#           ││└─ group: rwx
#           │└─ owner: rwx
#           └─ setgid: new files inherit group

# Step 3: Add team members
$ sudo usermod -aG dev,deploy alice    # Developer + deploy
$ sudo usermod -aG dev bob             # Developer only
$ sudo usermod -aG ops,deploy sam      # DevOps + deploy

# Step 4: Verify
$ ls -la /var/www/project
drwxrwsr-x 2 root deploy 4096 Jan 31 10:00 .
#     │
#     └─ setgid bit: new files will be owned by 'deploy' group
```

**BEFORE:**
```
alice tries to deploy:
$ touch /var/www/project/app.py
touch: cannot touch '/var/www/project/app.py': Permission denied
```

**AFTER (alice in deploy group):**
```
$ touch /var/www/project/app.py
$ ls -l /var/www/project/app.py
-rw-rw-r-- 1 alice deploy 0 Jan 31 10:00 app.py
#                  └─ Inherited from setgid directory!
```

---

## 🔐 **SUDO - CONTROLLED ROOT ACCESS**

### **What is sudo?**

`sudo` = "**s**ubstitute **u**ser **do**" (or "superuser do")

Allows regular users to run specific commands as root (or another user) **with accountability**.

```
WITHOUT sudo:
┌─────────────────────────────────────────┐
│ alice needs to restart nginx            │
│                ↓                        │
│ "Hey admin, can you restart nginx?"     │
│                ↓                        │
│ Admin logs in as root, runs command     │
│                ↓                        │
│ No audit trail of WHO requested it      │
└─────────────────────────────────────────┘

WITH sudo:
┌─────────────────────────────────────────┐
│ alice needs to restart nginx            │
│                ↓                        │
│ alice runs: sudo systemctl restart nginx│
│                ↓                        │
│ Logged: "alice ran systemctl restart nginx"
│                ↓                        │
│ Full audit trail, accountability        │
└─────────────────────────────────────────┘
```

---

### **Basic sudo Usage**

```bash
# Run command as root
$ sudo systemctl restart nginx

# Run command as another user
$ sudo -u postgres psql
#      └─ run as postgres user

# Run shell as root
$ sudo -i
root@server:~#

# Run shell as another user
$ sudo -u alice bash

# Edit file as root
$ sudo nano /etc/nginx/nginx.conf
# OR better:
$ sudoedit /etc/nginx/nginx.conf
# (uses SUDO_EDITOR, safer)

# Check what you can run
$ sudo -l
User sam may run the following commands on server:
    (ALL : ALL) ALL
```

---

### **The sudoers File**

sudo configuration lives in `/etc/sudoers`. **NEVER edit directly!**

```bash
# ALWAYS use visudo (validates syntax)
$ sudo visudo
```

**Why visudo?**
- Locks file during editing
- Validates syntax before saving
- Syntax error in sudoers = **locked out of sudo!**

---

### **sudoers Syntax**

```
WHO    WHERE=(AS_WHO:AS_GROUP)    WHAT

sam    ALL=(ALL:ALL)              ALL
│      │    │   │                 │
│      │    │   │                 └─ Commands allowed
│      │    │   └─ Can run as any group
│      │    └─ Can run as any user
│      └─ On any host
└─ Username
```

---

### **Common sudoers Configurations**

#### **1. Full sudo Access (Admin)**

```sudoers
# User sam has full sudo access
sam    ALL=(ALL:ALL) ALL

# Same via group (better for multiple users)
%sudo  ALL=(ALL:ALL) ALL
#└─ % prefix means GROUP
```

#### **2. Specific Commands Only**

```sudoers
# alice can only restart nginx and apache
alice  ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl restart apache2

# bob can only view logs
bob    ALL=(root) /usr/bin/journalctl, /usr/bin/tail /var/log/*
```

#### **3. No Password Required**

```sudoers
# Deployment user doesn't need password for deploy script
deploy ALL=(root) NOPASSWD: /opt/scripts/deploy.sh

# WARNING: Use sparingly! Security risk
```

#### **4. Run as Specific User**

```sudoers
# sam can run commands as postgres (for DB maintenance)
sam    ALL=(postgres) /usr/bin/psql, /usr/bin/pg_dump
```

#### **5. Command Aliases (Grouping)**

```sudoers
# Define command groups
Cmnd_Alias WEB_CMDS = /usr/bin/systemctl restart nginx, \
                       /usr/bin/systemctl reload nginx, \
                       /usr/bin/systemctl status nginx

Cmnd_Alias DEPLOY_CMDS = /opt/scripts/deploy.sh, \
                          /usr/bin/git pull

# Apply to users/groups
%dev   ALL=(root) WEB_CMDS
%ops   ALL=(root) WEB_CMDS, DEPLOY_CMDS
```

---

### **Using Drop-in Files (Best Practice)**

Instead of editing `/etc/sudoers`, create files in `/etc/sudoers.d/`:

```bash
# Create deploy team permissions
$ sudo visudo -f /etc/sudoers.d/deploy-team
```

```sudoers
# /etc/sudoers.d/deploy-team
%deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart flask-app, \
                             /usr/bin/systemctl status flask-app
```

**Benefits:**
- Easy to add/remove
- Package updates won't overwrite
- Version control friendly

---

### **Real-World Example: Developer Permissions**

**Scenario:** Developers need to:
- Restart the Flask app
- View application logs
- NOT access database or other system functions

```bash
$ sudo visudo -f /etc/sudoers.d/developers
```

```sudoers
# /etc/sudoers.d/developers
# Developers can manage the Flask application

# Command aliases
Cmnd_Alias FLASK_CMDS = /usr/bin/systemctl start flask-app, \
                         /usr/bin/systemctl stop flask-app, \
                         /usr/bin/systemctl restart flask-app, \
                         /usr/bin/systemctl reload flask-app, \
                         /usr/bin/systemctl status flask-app

Cmnd_Alias LOG_CMDS = /usr/bin/journalctl -u flask-app*, \
                       /usr/bin/tail -f /var/log/flask-app/*

# Apply to dev group
%dev   ALL=(root) FLASK_CMDS, LOG_CMDS
```

**BEFORE:**
```bash
alice$ sudo systemctl restart flask-app
[sudo] password for alice:
Sorry, user alice is not allowed to execute '/bin/systemctl restart flask-app' as root.
```

**AFTER:**
```bash
alice$ sudo systemctl restart flask-app
[sudo] password for alice:
# Success! And logged in /var/log/auth.log
```

---

### **sudo Logging and Auditing**

All sudo usage is logged:

```bash
# View sudo logs
$ sudo grep sudo /var/log/auth.log
Jan 31 10:00:00 server sudo: alice : TTY=pts/0 ; PWD=/home/alice ; USER=root ; COMMAND=/usr/bin/systemctl restart flask-app
Jan 31 10:05:00 server sudo: bob : TTY=pts/1 ; PWD=/home/bob ; USER=root ; COMMAND=/usr/bin/tail -f /var/log/flask-app/error.log

# Failed attempts
$ sudo grep "NOT allowed" /var/log/auth.log
Jan 31 10:10:00 server sudo: bob : command not allowed ; TTY=pts/1 ; PWD=/home/bob ; USER=root ; COMMAND=/usr/bin/passwd alice
```

---

## 🔄 **SWITCHING USERS**

### **su - Switch User**

```bash
# Switch to root (need root password)
$ su -
Password:
root@server:~#

# Switch to another user
$ su - alice
Password:  # alice's password
alice@server:~$

# Run single command as another user
$ su - alice -c "whoami"
Password:
alice
```

**The `-` is important:**
```bash
# WITHOUT - (keeps current environment)
$ su alice
$ pwd
/home/sam  ← Still in sam's directory!
$ echo $HOME
/home/sam  ← Wrong HOME!

# WITH - (login shell, fresh environment)
$ su - alice
$ pwd
/home/alice  ← Correct!
$ echo $HOME
/home/alice  ← Correct!
```

---

### **sudo vs su**

| Aspect | sudo | su |
|--------|------|-----|
| Password needed | YOUR password | TARGET user's password |
| Logging | Full audit trail | Minimal |
| Granularity | Can limit commands | All or nothing |
| Best practice | ✅ Preferred | ⚠️ Avoid for root |

**Production recommendation:**
```bash
# Good: Use sudo
$ sudo systemctl restart nginx

# Avoid: Using su to root
$ su -
# (No audit of what root did!)
```

---

## 🛡️ **SECURITY BEST PRACTICES**

### **1. Disable Root Login**

```bash
# Check current status
$ sudo passwd -S root
root L 01/01/2026 0 99999 7 -1
#    └─ L = Locked

# Lock root password
$ sudo passwd -l root

# Disable root SSH login
$ sudo nano /etc/ssh/sshd_config
PermitRootLogin no

$ sudo systemctl restart sshd
```

---

### **2. Use Groups for Permissions**

```bash
# Bad: Individual permissions
$ sudo chown alice /var/www/app
$ sudo chown bob /var/www/app  # Now alice lost access!

# Good: Group permissions
$ sudo chown root:deploy /var/www/app
$ sudo usermod -aG deploy alice
$ sudo usermod -aG deploy bob
# Both have access, easy to add/remove team members
```

---

### **3. Principle of Least Privilege**

```bash
# Bad: Give full sudo
%dev ALL=(ALL) ALL

# Good: Only what's needed
%dev ALL=(root) /usr/bin/systemctl restart flask-app
```

---

### **4. Regular Audits**

```bash
# List all users with sudo access
$ grep -r "ALL=" /etc/sudoers /etc/sudoers.d/ 2>/dev/null

# Find users in sudo group
$ getent group sudo
sudo:x:27:sam

# List all UID 0 users (should only be root!)
$ awk -F: '$3 == 0 {print $1}' /etc/passwd
root

# Find accounts with no password
$ sudo awk -F: '$2 == "" {print $1}' /etc/shadow
```

---

### **5. Service Account Best Practices**

```bash
# Create proper service account
$ sudo useradd -r \
    -s /usr/sbin/nologin \
    -d /var/lib/myapp \
    -c "MyApp Service Account" \
    myapp

# No home directory clutter, can't login interactively
```

---

## 🛠️ **PRACTICAL PRODUCTION SCENARIOS**

### **Scenario 1: New Developer Onboarding**

```bash
# Step 1: Create user
$ sudo useradd -m -s /bin/bash -c "Alice Smith" alice

# Step 2: Set temporary password
$ sudo passwd alice
# Enter temporary password

# Step 3: Force password change on first login
$ sudo chage -d 0 alice

# Step 4: Add to appropriate groups
$ sudo usermod -aG dev,deploy alice

# Step 5: Set up SSH key (on alice's first login)
alice$ mkdir -p ~/.ssh
alice$ chmod 700 ~/.ssh
alice$ nano ~/.ssh/authorized_keys  # paste public key
alice$ chmod 600 ~/.ssh/authorized_keys

# Step 6: Verify
$ id alice
uid=1001(alice) gid=1001(alice) groups=1001(alice),1001(dev),1003(deploy)
```

---

### **Scenario 2: Developer Offboarding**

```bash
# Step 1: Disable account immediately
$ sudo usermod -L alice

# Step 2: Kill active sessions
$ sudo pkill -u alice

# Step 3: Check what they own
$ sudo find /var/www -user alice -ls

# Step 4: Reassign ownership
$ sudo chown -R bob:dev /var/www/app/alice_projects/

# Step 5: Backup home directory
$ sudo tar -czf /backup/alice_home_$(date +%Y%m%d).tar.gz /home/alice

# Step 6: Remove from groups
$ sudo gpasswd -d alice dev
$ sudo gpasswd -d alice deploy

# Step 7: Delete account (after retention period)
$ sudo userdel -r alice
```

---

### **Scenario 3: "Who Restarted the Server?"**

```bash
# Check auth log for sudo commands
$ sudo grep "sudo:" /var/log/auth.log | grep -i reboot
Jan 31 03:00:00 server sudo: bob : TTY=pts/0 ; PWD=/home/bob ; USER=root ; COMMAND=/sbin/reboot

# Found it! Bob rebooted at 3 AM

# Check who was logged in at that time
$ last | grep "Jan 31"
bob      pts/0    192.168.1.50     Fri Jan 31 02:55 - 03:05  (00:10)
```

---

### **Scenario 4: Emergency Access Recovery**

```bash
# Problem: Locked out of sudo (sudoers syntax error)
# Solution: Boot to recovery mode or use console

# If you have console access:
# 1. Reboot, hold SHIFT for GRUB menu
# 2. Select "Advanced options" → "Recovery mode"
# 3. Select "root shell"

root# visudo
# Fix syntax error
root# exit
# Resume normal boot
```

---

## 🧪 **MINI-TASK: User & Access Management**

### **Objective:**
Set up a realistic team structure with proper permissions and sudo access.

---

### **Part 1: Create Team Structure**

```bash
# Create groups
sudo groupadd webdev
sudo groupadd dbadmin
sudo groupadd deployers

# Create shared directory
sudo mkdir -p /opt/webapp
```

---

### **Part 2: Create Users**

Create these users with proper settings:

| Username | Full Name | Groups | Shell | Notes |
|----------|-----------|--------|-------|-------|
| `alice` | Alice Developer | webdev, deployers | bash | Developer |
| `bob` | Bob Backend | webdev | bash | Developer (no deploy) |
| `carol` | Carol DBA | dbadmin | bash | Database admin |
| `appuser` | - | - | nologin | Service account |

---

### **Part 3: Configure sudo Access**

Create `/etc/sudoers.d/webapp-team` with:

1. **webdev group** can:
   - Restart/reload/status `myapp.service` (from Module 4)

2. **deployers group** can:
   - All webdev permissions
   - Write to `/opt/webapp/` (via proper group ownership, not sudo)

3. **dbadmin group** can:
   - Run `psql` and `pg_dump` as postgres user

---

### **Part 4: Test and Verify**

1. Login as each user and test permissions
2. Verify alice CAN restart myapp
3. Verify bob CANNOT deploy to /opt/webapp
4. Verify carol CAN run psql as postgres
5. Verify appuser CANNOT login

---

### **Part 5: Create Report**

Create `/home/sam/DevopsRoadmap/Phase-1-Linux/users-report.txt` containing:

1. Output of `cat /etc/group | grep -E "webdev|dbadmin|deployers"`
2. Output of `id alice`, `id bob`, `id carol`
3. Your sudoers.d/webapp-team file content
4. Proof that alice can restart myapp (sudo output)
5. Proof that bob cannot deploy (permission denied)
6. Proof that carol can run psql as postgres

---

### **Security Questions:**

1. What's the danger of `usermod -G` without `-a`?
2. Why use `/etc/sudoers.d/` instead of editing `/etc/sudoers`?
3. Why should service accounts have `/usr/sbin/nologin` as shell?
4. How would you find all users who have sudo access?

---

**Expected Time:** 60-90 minutes

**Deliverable:** Show me your `users-report.txt` and explain what you learned.

---

## 📌 **Quick Reference Card**

```bash
# User Information
whoami                      # Current username
id                          # Current user's UID, GID, groups
id alice                    # Another user's info
who / w                     # Who's logged in
groups alice                # User's groups

# User Management
sudo useradd -m -s /bin/bash -c "Full Name" username
sudo passwd username        # Set password
sudo usermod -aG group user # Add to group (DON'T FORGET -a!)
sudo userdel -r username    # Delete user and home

# Group Management
sudo groupadd groupname     # Create group
sudo groupdel groupname     # Delete group
sudo gpasswd -a user group  # Add user to group
sudo gpasswd -d user group  # Remove user from group

# Password Management
sudo passwd -e user         # Force password change
sudo passwd -l user         # Lock account
sudo passwd -u user         # Unlock account
sudo chage -l user          # View password aging

# sudo
sudo command                # Run as root
sudo -u user command        # Run as specific user
sudo -l                     # List your permissions
sudo visudo                 # Edit sudoers (safely)
sudo visudo -f /etc/sudoers.d/file  # Edit drop-in file

# Switch User
su - username               # Switch user (login shell)
sudo -i                     # Root shell via sudo

# Files
/etc/passwd                 # User database
/etc/shadow                 # Password hashes
/etc/group                  # Group database
/etc/sudoers                # sudo configuration
/etc/sudoers.d/             # sudo drop-in files
```

---

**Complete this mini-task, then we'll move to Module 6: Package Management.**
