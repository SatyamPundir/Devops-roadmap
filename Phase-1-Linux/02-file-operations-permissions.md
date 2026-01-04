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

## 🔐 Linux Permission Model

Every file/directory has:
1. **Owner** (user)
2. **Group** (group of users)
3. **Others** (everyone else)

Each can have:
- **r** (read) - view contents
- **w** (write) - modify contents
- **x** (execute) - run as program / enter directory

### **Understanding `ls -l` Output**

```bash
$ ls -l /var/log/nginx/access.log
-rw-r--r-- 1 www-data www-data 12345 Jan 4 10:30 access.log
│├─┬─┬─┬─┤ │ │        │        │     │         │
│││││││││└─ Permissions for Others
││││││││└── Permissions for Group  
│││││││└─── Permissions for Owner
││││││└──── Type + Permissions
│││││└───── Owner (user)
││││└────── Group
│││└─────── Size (bytes)
││└──────── Modification date
│└───────── Filename
└────────── File type (- = file, d = directory, l = symlink)
```

### **File Type Indicators**

| Symbol | Type |
|--------|------|
| `-` | Regular file |
| `d` | Directory |
| `l` | Symbolic link |
| `c` | Character device |
| `b` | Block device |
| `s` | Socket |
| `p` | Named pipe |

### **Permission Breakdown**

```
-rw-r--r--
│││ │ │ │
│││ │ │ └── Others: read only (r--)
│││ │ └──── Group: read only (r--)
│││ └────── Owner: read + write (rw-)
││└──────── Type: regular file (-)
│└───────── (These 3 positions are for owner)
└────────── File type
```

### **Permission Values**

| Permission | Symbol | Numeric | Meaning for Files | Meaning for Directories |
|------------|--------|---------|-------------------|-------------------------|
| Read | r | 4 | View file contents | List directory contents |
| Write | w | 2 | Modify file | Create/delete files inside |
| Execute | x | 1 | Run as program | Enter directory (cd into it) |
| None | - | 0 | No permission | No permission |

### **Numeric Permission Calculation**

```
rwx = 4 + 2 + 1 = 7
rw- = 4 + 2 + 0 = 6
r-x = 4 + 0 + 1 = 5
r-- = 4 + 0 + 0 = 4
-wx = 0 + 2 + 1 = 3
-w- = 0 + 2 + 0 = 2
--x = 0 + 0 + 1 = 1
--- = 0 + 0 + 0 = 0
```

**Common Permission Patterns**:

| Numeric | Symbolic | Use Case |
|---------|----------|----------|
| 755 | rwxr-xr-x | Scripts, binaries (owner can modify, others can read/execute) |
| 644 | rw-r--r-- | Config files, documents (owner can write, others read only) |
| 600 | rw------- | Private files, SSH keys (only owner can read/write) |
| 700 | rwx------ | Private directories (only owner can access) |
| 775 | rwxrwxr-x | Shared directories (group can write) |
| 666 | rw-rw-rw- | World-writable files (DANGEROUS - avoid) |
| 777 | rwxrwxrwx | World-writable/executable (VERY DANGEROUS - never use) |

---

## 🛠️ Changing Permissions

### **chmod** - Change Mode (Permissions)

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

## 👤 Changing Ownership

### **chown** - Change Owner

```bash
chown user file.txt                   # Change owner
chown user:group file.txt             # Change owner and group
chown :group file.txt                 # Change group only
chown -R user:group /path/dir/        # Recursive
```

**Production Examples**:
```bash
# After deploying application
sudo chown -R www-data:www-data /var/www/html/

# Fix Docker socket permissions
sudo chown root:docker /var/run/docker.sock

# Application ownership
sudo chown -R appuser:appuser /opt/myapp/
```

### **chgrp** - Change Group (alternative to chown)

```bash
chgrp group file.txt                  # Change group
chgrp -R group /path/dir/             # Recursive
```

---

## 🔒 Special Permissions

### **setuid (Set User ID) - 4xxx**

When executed, runs with **owner's permissions**, not executor's.

```bash
chmod u+s /path/to/binary             # Symbolic
chmod 4755 /path/to/binary            # Numeric

# Example: /usr/bin/passwd
ls -l /usr/bin/passwd
# -rwsr-xr-x (s instead of x for owner)
```

**Why it exists**: `passwd` command needs root permissions to modify `/etc/shadow`, but regular users must be able to change their password.

**Security Risk**: Vulnerable if binary has bugs (privilege escalation).

### **setgid (Set Group ID) - 2xxx**

For files: Runs with file's group permissions.  
For directories: New files inherit directory's group (not creator's group).

```bash
chmod g+s /shared/dir/                # Symbolic
chmod 2775 /shared/dir/               # Numeric
```

**Production Use**:
```bash
# Shared project directory
mkdir /opt/shared-project
chgrp developers /opt/shared-project
chmod 2775 /opt/shared-project
# Now all files created inside inherit "developers" group
```

### **Sticky Bit - 1xxx**

On directories: Only file owner can delete their files (even if others have write permission).

```bash
chmod +t /tmp/                        # Symbolic
chmod 1777 /tmp/                      # Numeric
```

**Production Use**: `/tmp` directory has sticky bit so users can't delete each other's temp files.

```bash
ls -ld /tmp
# drwxrwxrwt (t instead of x for others)
```

---

## 🎭 umask - Default Permissions

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
mv source dest              # Move/rename
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

**Complete this mini-task, then we'll move to Module 3: Process Management.**
