# Module 7: Text Processing

## 🎯 Why This Matters in Production

**Scenario**: It's 3 AM. Alerts firing. Your application is returning 500 errors. You have 50GB of logs across multiple servers.

You need to:
- **Find all ERROR lines from the last hour**
- **Extract unique IP addresses causing failures**
- **Count how many times each error type occurred**
- **Transform config files during deployment**

**Text processing = your ability to investigate, troubleshoot, and automate.**

---

## 🧠 **THE TEXT PROCESSING PIPELINE**

Linux follows the **Unix philosophy**: small tools that do ONE thing well, connected via pipes.

```
┌────────────────────────────────────────────────────────────────────────┐
│                     TEXT PROCESSING PIPELINE                           │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   INPUT          FILTER         TRANSFORM       AGGREGATE      OUTPUT  │
│                                                                         │
│  ┌─────┐       ┌─────────┐     ┌─────────┐     ┌─────────┐    ┌─────┐ │
│  │ cat │──────▶│  grep   │────▶│   sed   │────▶│  sort   │───▶│file │ │
│  │ log │       │  awk    │     │   awk   │     │  uniq   │    │     │ │
│  │ file│       │  head   │     │   cut   │     │  wc     │    │stdout│ │
│  └─────┘       │  tail   │     │   tr    │     └─────────┘    └─────┘ │
│                └─────────┘     └─────────┘                             │
│                                                                         │
│   Example: cat access.log | grep "500" | awk '{print $1}' | sort -u    │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 📖 **GREP - Global Regular Expression Print**

### **What is grep?**

`grep` searches for PATTERNS in files and outputs matching lines.

```
┌──────────────────────────────────────────────────────────────┐
│  grep [OPTIONS] PATTERN [FILE...]                            │
│                                                              │
│  Think of it as: "Show me all lines containing X"            │
└──────────────────────────────────────────────────────────────┘
```

### **Basic Usage**

```bash
# Search for a string in a file
grep "error" /var/log/syslog

# Search in multiple files
grep "error" /var/log/*.log

# Search recursively in directories
grep -r "TODO" /home/sam/project/

# Case-insensitive search
grep -i "error" /var/log/syslog
```

### **Essential grep Options**

| Option | Meaning | Example |
|--------|---------|---------|
| `-i` | Case insensitive | `grep -i "error" log` |
| `-v` | Invert match (NOT) | `grep -v "debug" log` |
| `-n` | Show line numbers | `grep -n "error" log` |
| `-c` | Count matches | `grep -c "error" log` |
| `-l` | List filenames only | `grep -l "error" *.log` |
| `-L` | List files WITHOUT match | `grep -L "error" *.log` |
| `-r` | Recursive search | `grep -r "TODO" src/` |
| `-w` | Match whole words | `grep -w "error" log` |
| `-A N` | Show N lines AFTER | `grep -A 3 "error" log` |
| `-B N` | Show N lines BEFORE | `grep -B 3 "error" log` |
| `-C N` | Show N lines CONTEXT | `grep -C 3 "error" log` |
| `-o` | Only matching part | `grep -o "IP=[0-9.]*"` |
| `-E` | Extended regex (ERE) | `grep -E "error|warn"` |
| `-P` | Perl regex (PCRE) | `grep -P "\d{3}"` |

### **grep with Regular Expressions**

```
┌─────────────────────────────────────────────────────────────────────┐
│              BASIC REGULAR EXPRESSION PATTERNS                       │
├─────────────────────────────────────────────────────────────────────┤
│  .        Any single character       grep "h.t" → hat, hit, hot    │
│  *        Zero or more of previous   grep "go*d" → gd, god, good   │
│  ^        Start of line              grep "^Error" → lines starting│
│  $        End of line                grep "fail$" → lines ending   │
│  []       Character class            grep "[Ee]rror" → Error, error│
│  [^]      Negated class              grep "[^0-9]" → non-digits    │
│  \        Escape special char        grep "\." → literal dot       │
├─────────────────────────────────────────────────────────────────────┤
│              EXTENDED REGEX (-E or egrep)                           │
├─────────────────────────────────────────────────────────────────────┤
│  +        One or more                grep -E "go+d" → god, good    │
│  ?        Zero or one                grep -E "colou?r" → color/our │
│  |        OR (alternation)           grep -E "cat|dog"             │
│  ()       Grouping                   grep -E "(ab)+" → ab, abab    │
│  {n}      Exactly n times            grep -E "[0-9]{3}" → 3 digits │
│  {n,m}    Between n and m times      grep -E "[0-9]{2,4}"          │
└─────────────────────────────────────────────────────────────────────┘
```

### **Production grep Examples**

```bash
# Find all 5xx errors in nginx log
grep -E "HTTP/[0-9.]+ 5[0-9]{2}" /var/log/nginx/access.log

# Find errors in last 1000 lines
tail -1000 /var/log/app.log | grep -i "error"

# Find IP addresses making failed logins
grep "Failed password" /var/log/auth.log | grep -oE "[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+"

# Find all TODO/FIXME in codebase
grep -rn -E "TODO|FIXME" --include="*.py" /app/

# Find config lines (ignore comments and blank)
grep -v "^#" /etc/nginx/nginx.conf | grep -v "^$"

# Count errors per log file
grep -c "ERROR" /var/log/app/*.log

# Show errors with 5 lines of context
grep -C 5 "Exception" /var/log/app.log

# Find processes listening on ports
netstat -tlnp 2>/dev/null | grep -E ":[0-9]+ "
```

---

## ✂️ **CUT - Extract Columns/Fields**

### **What is cut?**

`cut` extracts specific columns or fields from each line.

```
┌──────────────────────────────────────────────────────────────┐
│  cut -d'DELIMITER' -f FIELDS [FILE]                          │
│                                                              │
│  Think of it as: "Give me column X from this data"           │
└──────────────────────────────────────────────────────────────┘
```

### **cut Options**

| Option | Meaning | Example |
|--------|---------|---------|
| `-d` | Field delimiter | `-d':'` for colon-separated |
| `-f` | Field numbers | `-f1,3` or `-f1-3` |
| `-c` | Character positions | `-c1-10` first 10 chars |
| `--complement` | Invert selection | All EXCEPT specified |

### **cut Examples**

```bash
# /etc/passwd format: username:x:uid:gid:comment:home:shell
#                     field 1  2  3   4     5     6    7

# Extract usernames (field 1)
cut -d':' -f1 /etc/passwd

# Extract username and shell (fields 1 and 7)
cut -d':' -f1,7 /etc/passwd

# Extract username through home directory (fields 1-6)
cut -d':' -f1-6 /etc/passwd

# Extract first 10 characters of each line
cut -c1-10 /etc/passwd

# From CSV: extract columns 2 and 4
cut -d',' -f2,4 data.csv

# From log: extract timestamp (first 15 chars)
cut -c1-15 /var/log/syslog

# Combine with grep
grep "sam" /etc/passwd | cut -d':' -f1,6,7
```

### **Production cut Examples**

```bash
# Extract IPs from access log (space-delimited, field 1)
cut -d' ' -f1 /var/log/nginx/access.log

# Get process names from ps output
ps aux | cut -c66-

# Extract date from log entries
grep "ERROR" app.log | cut -d' ' -f1-2

# Parse CSV exports
cut -d',' -f1,3,5 users_export.csv
```

---

## 📊 **AWK - Pattern Scanning and Processing**

### **What is awk?**

`awk` is a full programming language for text processing. It processes text line by line, automatically splitting into fields.

```
┌──────────────────────────────────────────────────────────────┐
│  awk 'PATTERN { ACTION }' [FILE]                             │
│                                                              │
│  For each line:                                              │
│    1. Split line into fields ($1, $2, ... $NF)              │
│    2. If PATTERN matches, execute ACTION                     │
│                                                              │
│  $0 = entire line    $1 = first field    $NF = last field   │
└──────────────────────────────────────────────────────────────┘
```

### **awk Built-in Variables**

| Variable | Meaning |
|----------|---------|
| `$0` | Entire current line |
| `$1, $2...` | Field 1, Field 2, etc. |
| `$NF` | Last field |
| `NF` | Number of fields |
| `NR` | Current line number (record) |
| `FS` | Field separator (default: space) |
| `OFS` | Output field separator |
| `RS` | Record separator (default: newline) |

### **Basic awk Usage**

```bash
# Print entire line
awk '{print $0}' file

# Print first field
awk '{print $1}' file

# Print first and third fields
awk '{print $1, $3}' file

# Print with custom separator
awk '{print $1 " -> " $3}' file

# Print last field
awk '{print $NF}' file

# Print second-to-last field
awk '{print $(NF-1)}' file

# Custom field separator
awk -F':' '{print $1, $7}' /etc/passwd

# Multiple separators
awk -F'[:@]' '{print $1}' file
```

### **awk with Patterns**

```bash
# Lines containing "error"
awk '/error/' file

# Lines NOT containing "debug"
awk '!/debug/' file

# Lines where field 3 > 100
awk '$3 > 100' file

# Lines where field 1 equals "sam"
awk '$1 == "sam"' file

# Line numbers 5-10
awk 'NR >= 5 && NR <= 10' file

# First line only
awk 'NR == 1' file

# Last line (requires reading entire file)
awk 'END {print}' file
```

### **awk with BEGIN and END**

```bash
# BEGIN: runs before processing any lines
# END: runs after processing all lines

# Add header and footer
awk 'BEGIN {print "=== USERS ==="} {print $1} END {print "=== END ==="}' /etc/passwd

# Count lines
awk 'END {print NR " lines"}' file

# Sum a column
awk '{sum += $3} END {print "Total:", sum}' data.txt

# Calculate average
awk '{sum += $3; count++} END {print "Average:", sum/count}' data.txt
```

### **awk Formatting**

```bash
# printf for formatted output
awk '{printf "%-20s %10d\n", $1, $3}' file
#     %-20s = left-aligned string, 20 chars
#     %10d = right-aligned integer, 10 chars

# Common format specifiers
# %s   string
# %d   integer
# %f   floating point
# %.2f floating point, 2 decimal places
# %-   left align
# %10  minimum width 10
```

### **Production awk Examples**

```bash
# Sum of response times from access log
awk '{sum += $NF} END {print "Total:", sum, "Avg:", sum/NR}' access.log

# Top 10 IPs by request count
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# Find requests taking > 5 seconds
awk '$NF > 5 {print $0}' access.log

# Calculate disk usage percentage
df -h | awk 'NR>1 {print $5, $6}' | sort -rn

# Parse /etc/passwd for users with bash shell
awk -F':' '$7 ~ /bash/ {print $1, $6}' /etc/passwd

# Memory usage by process
ps aux | awk 'NR>1 {mem[$11] += $6} END {for (p in mem) print p, mem[p]}' | sort -k2 -rn | head

# Convert Apache log to CSV
awk '{print $1","$4","$7","$9}' access.log > access.csv

# Find 5xx errors with timestamps
awk '$9 ~ /^5/ {print $4, $7, $9}' access.log
```

---

## 🔄 **SED - Stream Editor**

### **What is sed?**

`sed` performs text transformations on streams (files or input). It's primarily used for search-and-replace operations.

```
┌──────────────────────────────────────────────────────────────┐
│  sed [OPTIONS] 'COMMAND' [FILE]                              │
│                                                              │
│  Most common: sed 's/OLD/NEW/g' file                         │
│               s = substitute                                 │
│               g = global (all occurrences)                   │
└──────────────────────────────────────────────────────────────┘
```

### **sed Substitution**

```bash
# Basic substitution (first occurrence per line)
sed 's/error/ERROR/' file

# Global substitution (all occurrences)
sed 's/error/ERROR/g' file

# Case-insensitive substitution
sed 's/error/ERROR/gi' file

# Edit file in-place
sed -i 's/old/new/g' file

# In-place with backup
sed -i.bak 's/old/new/g' file

# Delete matched pattern
sed 's/pattern//g' file

# Different delimiters (useful for paths)
sed 's|/old/path|/new/path|g' file
sed 's#/old/path#/new/path#g' file
```

### **sed Line Operations**

```bash
# Delete lines containing pattern
sed '/pattern/d' file

# Delete blank lines
sed '/^$/d' file

# Delete lines 5-10
sed '5,10d' file

# Print only lines 5-10
sed -n '5,10p' file

# Delete first line
sed '1d' file

# Delete last line
sed '$d' file

# Insert line before pattern
sed '/pattern/i\New line before' file

# Insert line after pattern
sed '/pattern/a\New line after' file

# Replace entire line matching pattern
sed '/pattern/c\Replacement line' file
```

### **sed with Regex and Groups**

```bash
# Capture groups: \1, \2, etc.
# Swap first two words
sed 's/\(\w*\) \(\w*\)/\2 \1/' file

# Extended regex (easier grouping)
sed -E 's/(\w+) (\w+)/\2 \1/' file

# Add prefix to lines
sed 's/^/PREFIX: /' file

# Add suffix to lines
sed 's/$/ :SUFFIX/' file

# Remove leading whitespace
sed 's/^[ \t]*//' file

# Remove trailing whitespace
sed 's/[ \t]*$//' file

# Remove both leading and trailing whitespace
sed 's/^[ \t]*//;s/[ \t]*$//' file
```

### **Production sed Examples**

```bash
# Update config file values
sed -i 's/^DEBUG=.*/DEBUG=false/' /etc/app.conf

# Comment out a line
sed -i 's/^dangerous_option/#&/' config

# Uncomment a line
sed -i 's/^#\(wanted_option\)/\1/' config

# Update database connection string
sed -i 's/localhost:5432/db.prod.internal:5432/g' app.config

# Remove Windows carriage returns
sed -i 's/\r$//' script.sh

# Add timestamp to each log line
sed 's/^/[2026-02-07] /' input.log > output.log

# Mask sensitive data in logs
sed 's/password=.*/password=REDACTED/g' app.log

# Change file paths in configs during deployment
sed -i 's|/home/dev/app|/opt/app|g' *.conf

# Extract content between markers
sed -n '/START/,/END/p' file
```

---

## 📈 **SORT - Sort Lines**

### **What is sort?**

`sort` arranges lines in a specified order.

```
┌──────────────────────────────────────────────────────────────┐
│  sort [OPTIONS] [FILE]                                       │
│                                                              │
│  Default: alphabetical sort of entire line                   │
└──────────────────────────────────────────────────────────────┘
```

### **sort Options**

| Option | Meaning | Example |
|--------|---------|---------|
| `-n` | Numeric sort | Sort numbers correctly |
| `-r` | Reverse order | Descending |
| `-k N` | Sort by field N | `-k 2` sort by column 2 |
| `-t` | Field delimiter | `-t':'` for colon |
| `-u` | Unique (remove dupes) | Sort + dedupe |
| `-f` | Case insensitive | Fold case |
| `-h` | Human numeric (1K, 2M) | For sizes |
| `-V` | Version sort | 1.2 < 1.10 |
| `-o` | Output to file | `-o sorted.txt` |

### **sort Examples**

```bash
# Alphabetical sort
sort names.txt

# Numeric sort
sort -n numbers.txt

# Reverse sort
sort -r names.txt

# Sort by second column
sort -k2 data.txt

# Sort by second column numerically
sort -k2 -n data.txt

# Sort /etc/passwd by UID (field 3)
sort -t':' -k3 -n /etc/passwd

# Sort and remove duplicates
sort -u names.txt

# Sort by file size (human readable)
ls -lh | sort -k5 -h

# Sort IP addresses
sort -t'.' -k1,1n -k2,2n -k3,3n -k4,4n ips.txt

# Sort by multiple fields
sort -k1,1 -k2,2n data.txt
```

### **Production sort Examples**

```bash
# Top 10 largest files
du -ah /var/log | sort -rh | head -10

# Sort processes by memory
ps aux | sort -k4 -rn | head -10

# Sort access log by response time (last field)
sort -k NF -n access.log | tail -20

# Sort unique IPs by frequency
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head
```

---

## 🔢 **UNIQ - Report or Filter Repeated Lines**

### **What is uniq?**

`uniq` filters ADJACENT duplicate lines. **Important**: Data must be sorted first!

```
┌──────────────────────────────────────────────────────────────┐
│  uniq [OPTIONS] [INPUT [OUTPUT]]                             │
│                                                              │
│  ⚠️ CRITICAL: uniq only works on ADJACENT duplicates!       │
│     ALWAYS: sort file | uniq                                 │
└──────────────────────────────────────────────────────────────┘
```

### **uniq Options**

| Option | Meaning | Example |
|--------|---------|---------|
| `-c` | Count occurrences | Show count prefix |
| `-d` | Only duplicates | Lines appearing > 1 |
| `-u` | Only unique | Lines appearing = 1 |
| `-i` | Case insensitive | Ignore case |
| `-f N` | Skip first N fields | Compare from field N+1 |
| `-s N` | Skip first N chars | Compare from char N+1 |

### **uniq Examples**

```bash
# Remove adjacent duplicates
uniq file.txt

# Count occurrences (MUST sort first!)
sort file.txt | uniq -c

# Show only duplicated lines
sort file.txt | uniq -d

# Show only unique lines (appear once)
sort file.txt | uniq -u

# Case-insensitive duplicate removal
sort -f file.txt | uniq -i

# Top 10 most common entries
sort file.txt | uniq -c | sort -rn | head -10
```

### **Production uniq Examples**

```bash
# Top 10 IP addresses in access log
cut -d' ' -f1 access.log | sort | uniq -c | sort -rn | head -10

# Most common HTTP response codes
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Find duplicate files (by checksum)
md5sum * | sort | uniq -d -w32

# Users who logged in more than once
last | awk '{print $1}' | sort | uniq -c | awk '$1 > 1'

# Unique error messages
grep ERROR app.log | cut -d']' -f2 | sort | uniq -c | sort -rn
```

---

## 📏 **WC - Word Count**

### **What is wc?**

`wc` counts lines, words, and characters.

```bash
# Default: lines, words, characters
wc file.txt
# Output: 42  156  1024 file.txt
#         lines words bytes

# Lines only
wc -l file.txt

# Words only
wc -w file.txt

# Characters only
wc -c file.txt

# Multiple files
wc -l *.log
```

### **Production wc Examples**

```bash
# Count error occurrences
grep -c "ERROR" app.log
# OR
grep "ERROR" app.log | wc -l

# Count active connections
netstat -an | grep ESTABLISHED | wc -l

# Count files in directory
ls -1 /var/log | wc -l

# Count lines of code
find . -name "*.py" -exec cat {} \; | wc -l
# Better: exclude blank lines
find . -name "*.py" -exec cat {} \; | grep -v "^$" | wc -l
```

---

## 🔀 **TR - Translate Characters**

### **What is tr?**

`tr` translates or deletes characters.

```
┌──────────────────────────────────────────────────────────────┐
│  tr [OPTIONS] SET1 [SET2]                                    │
│                                                              │
│  Reads from stdin, writes to stdout                          │
│  Does NOT accept filenames - use with pipes/redirection      │
└──────────────────────────────────────────────────────────────┘
```

### **tr Examples**

```bash
# Convert lowercase to uppercase
echo "hello" | tr 'a-z' 'A-Z'
# HELLO

# Convert uppercase to lowercase
echo "HELLO" | tr 'A-Z' 'a-z'
# hello

# Delete characters
echo "hello123" | tr -d '0-9'
# hello

# Squeeze repeated characters
echo "hellooooo" | tr -s 'o'
# hello

# Replace spaces with newlines
echo "one two three" | tr ' ' '\n'

# Delete all non-printable characters
cat file | tr -cd '[:print:]\n'

# Convert Windows line endings
cat windows.txt | tr -d '\r' > unix.txt

# Replace multiple spaces with single space
echo "too    many    spaces" | tr -s ' '
```

### **tr Character Classes**

| Class | Meaning |
|-------|---------|
| `[:alpha:]` | Letters |
| `[:digit:]` | Digits |
| `[:alnum:]` | Letters + digits |
| `[:space:]` | Whitespace |
| `[:upper:]` | Uppercase |
| `[:lower:]` | Lowercase |
| `[:print:]` | Printable chars |

---

## 🔗 **HEAD and TAIL - View Start/End of Files**

### **head - View Beginning**

```bash
# First 10 lines (default)
head file.txt

# First 20 lines
head -n 20 file.txt
head -20 file.txt

# All except last 5 lines
head -n -5 file.txt

# First 100 bytes
head -c 100 file.txt
```

### **tail - View End**

```bash
# Last 10 lines (default)
tail file.txt

# Last 50 lines
tail -n 50 file.txt
tail -50 file.txt

# Everything after line 10
tail -n +10 file.txt

# Follow file (live updates) - CRITICAL FOR LOGS
tail -f /var/log/syslog

# Follow with retry (if file rotates)
tail -F /var/log/syslog

# Follow multiple files
tail -f /var/log/*.log
```

### **Production head/tail Examples**

```bash
# Watch logs in real-time
tail -f /var/log/nginx/access.log | grep --line-buffered "ERROR"

# View recent errors with context
tail -1000 app.log | grep -C 3 "Exception"

# Skip header row in CSV
tail -n +2 data.csv | ...

# View log entries from last hour (approximately)
tail -10000 access.log | grep "$(date +%H:)"
```

---

## 🧪 **COMBINING TOOLS - Pipeline Power**

The real power of text processing comes from combining tools.

### **Example: Analyze Web Server Access Log**

```
Log format: IP - - [timestamp] "METHOD /path HTTP/1.1" status size
Example:   192.168.1.100 - - [07/Feb/2026:10:15:32] "GET /api/users HTTP/1.1" 200 1234
```

```bash
# 1. Top 10 IP addresses
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 2. Count requests per HTTP method
awk '{print $6}' access.log | tr -d '"' | sort | uniq -c | sort -rn

# 3. Find all 5xx errors with their URLs
awk '$9 ~ /^5/ {print $9, $7}' access.log | sort | uniq -c | sort -rn

# 4. Calculate total bandwidth served
awk '{sum += $10} END {print sum/1024/1024 " MB"}' access.log

# 5. Requests per hour
awk '{print $4}' access.log | cut -d: -f1,2 | uniq -c

# 6. Top 10 most requested pages
awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -10

# 7. IPs causing 404 errors
awk '$9 == 404 {print $1}' access.log | sort | uniq -c | sort -rn | head

# 8. Average response size per status code
awk '{count[$9]++; size[$9]+=$10} END {for(s in count) print s, size[s]/count[s]}' access.log
```

### **Example: System Troubleshooting**

```bash
# Find processes using most memory
ps aux | awk 'NR>1 {print $4, $11}' | sort -rn | head -10

# Find largest log files
find /var/log -type f -name "*.log" -exec du -h {} \; | sort -rh | head -10

# Count unique users logged in today
last | grep "$(date +%b\ %d)" | awk '{print $1}' | sort -u | wc -l

# Find failed SSH attempts
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn

# Check disk usage by directory
du -h --max-depth=1 /var | sort -rh

# Find files modified in last 24 hours
find /etc -type f -mtime -1 -exec ls -la {} \;
```

### **Example: Configuration Management**

```bash
# Update all config files with new server name
find /etc/app -name "*.conf" -exec sed -i 's/old-server/new-server/g' {} \;

# Extract all unique keys from config files
grep -h "^[a-zA-Z]" /etc/app/*.conf | cut -d'=' -f1 | sort -u

# Generate report of all enabled features
grep -r "enabled.*=.*true" /etc/app/ | awk -F: '{print $1, $2}'

# Validate no sensitive data in configs
grep -rE "(password|secret|key).*=.*[^$]" /etc/app/ --include="*.conf"
```

---

## 🏭 **PRODUCTION SCENARIOS**

### **Scenario 1: Security Incident - Find Attack Source**

```bash
# Situation: Someone is brute-forcing SSH

# Step 1: Find failed login attempts
grep "Failed password" /var/log/auth.log | tail -100

# Step 2: Extract attacker IPs
grep "Failed password" /var/log/auth.log | \
  grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | \
  sort | uniq -c | sort -rn | head -10

# Step 3: Find specific attacker's attempts over time
grep "192.168.1.50" /var/log/auth.log | \
  awk '{print $1, $2, $3}' | uniq -c

# Step 4: See what usernames they're trying
grep "Failed password" /var/log/auth.log | \
  grep "192.168.1.50" | \
  awk '{for(i=1;i<=NF;i++) if($i=="for") print $(i+1)}' | \
  sort | uniq -c | sort -rn
```

### **Scenario 2: Performance Issue - Slow API Endpoints**

```bash
# Access log format includes response time as last field

# Find slowest requests (> 5 seconds)
awk '$NF > 5 {print $NF, $7}' access.log | sort -rn | head -20

# Average response time by endpoint
awk '{time[$7] += $NF; count[$7]++} 
     END {for (e in time) printf "%.2f %s\n", time[e]/count[e], e}' access.log | \
  sort -rn | head -10

# Response time distribution
awk '{
  if ($NF < 0.1) bucket="<100ms"
  else if ($NF < 0.5) bucket="100-500ms"
  else if ($NF < 1) bucket="500ms-1s"
  else if ($NF < 5) bucket="1-5s"
  else bucket=">5s"
  count[bucket]++
} END {for (b in count) print count[b], b}' access.log | sort -rn
```

### **Scenario 3: Deployment - Config Updates**

```bash
# Update database host across all config files
find /opt/app -name "*.yml" -o -name "*.conf" | \
  xargs sed -i 's/db.staging.internal/db.prod.internal/g'

# Verify changes
grep -r "db.prod.internal" /opt/app/

# Create backup of all changed files
find /opt/app \( -name "*.yml" -o -name "*.conf" \) -newer /tmp/deploy_start \
  -exec cp {} {}.bak \;
```

### **Scenario 4: Log Rotation Check**

```bash
# Find largest log files
find /var/log -type f -size +100M 2>/dev/null | \
  xargs du -h | sort -rh

# Check log rotation status
ls -la /var/log/*.log* | head -20

# Find logs not rotated in 7 days
find /var/log -name "*.log" -mtime +7 -size +10M
```

---

## ⚠️ **COMMON MISTAKES**

### **1. uniq Without sort**

```bash
# WRONG - uniq only removes ADJACENT duplicates
cat file.txt | uniq

# RIGHT
sort file.txt | uniq
```

### **2. grep Without Anchors**

```bash
# WRONG - matches "error" anywhere: "errors", "myerror", etc.
grep "error" file

# BETTER - match word boundaries
grep -w "error" file

# OR use anchors
grep "^error$" file
```

### **3. sed In-Place Without Backup**

```bash
# DANGEROUS - no way to undo
sed -i 's/old/new/g' important.conf

# SAFE - creates backup
sed -i.bak 's/old/new/g' important.conf
```

### **4. Forgetting to Escape Special Characters**

```bash
# WRONG - . means "any character"
grep "192.168.1.1" file

# RIGHT - escape the dots
grep "192\.168\.1\.1" file

# OR use fixed strings
grep -F "192.168.1.1" file
```

### **5. tr Requires stdin**

```bash
# WRONG
tr 'a-z' 'A-Z' file.txt

# RIGHT
cat file.txt | tr 'a-z' 'A-Z'
# OR
tr 'a-z' 'A-Z' < file.txt
```

---

## 📝 **QUICK REFERENCE**

```bash
# ============== GREP ==============
grep "pattern" file              # Find lines with pattern
grep -i "pattern" file           # Case insensitive
grep -v "pattern" file           # Invert (NOT)
grep -c "pattern" file           # Count matches
grep -n "pattern" file           # Show line numbers
grep -r "pattern" dir/           # Recursive
grep -E "pat1|pat2" file         # Extended regex (OR)
grep -A3 -B3 "pattern" file      # Context lines

# ============== CUT ==============
cut -d':' -f1 file               # Field 1, colon delimiter
cut -d',' -f1,3 file             # Fields 1 and 3
cut -c1-10 file                  # Characters 1-10

# ============== AWK ==============
awk '{print $1}' file            # Print field 1
awk -F':' '{print $1}' file      # Custom delimiter
awk '$3 > 100' file              # Conditional
awk '/pattern/' file             # Pattern match
awk '{sum+=$1} END {print sum}'  # Sum column

# ============== SED ==============
sed 's/old/new/' file            # Replace first
sed 's/old/new/g' file           # Replace all
sed -i 's/old/new/g' file        # In-place edit
sed '/pattern/d' file            # Delete lines
sed -n '5,10p' file              # Print lines 5-10

# ============== SORT ==============
sort file                        # Alphabetical
sort -n file                     # Numeric
sort -r file                     # Reverse
sort -k2 file                    # By field 2
sort -t':' -k3 -n file           # Custom delimiter

# ============== UNIQ ==============
sort file | uniq                 # Remove duplicates
sort file | uniq -c              # Count occurrences
sort file | uniq -d              # Show only dupes

# ============== OTHERS ==============
wc -l file                       # Count lines
head -20 file                    # First 20 lines
tail -20 file                    # Last 20 lines
tail -f file                     # Follow live
tr 'a-z' 'A-Z' < file            # Translate chars
```

---

## 🎯 **MINI-TASK: Log Analysis Challenge**

Create a sample log file and perform analysis:

### **Part 1: Setup (Create Test Data)**

```bash
# Create a sample access log
cat << 'EOF' > /tmp/access.log
192.168.1.100 - - [07/Feb/2026:10:15:32] "GET /api/users HTTP/1.1" 200 1234 0.05
192.168.1.101 - - [07/Feb/2026:10:15:33] "POST /api/login HTTP/1.1" 200 89 0.12
192.168.1.100 - - [07/Feb/2026:10:15:34] "GET /api/users HTTP/1.1" 200 1234 0.03
192.168.1.102 - - [07/Feb/2026:10:15:35] "GET /api/products HTTP/1.1" 404 45 0.02
192.168.1.100 - - [07/Feb/2026:10:15:36] "GET /api/orders HTTP/1.1" 500 123 5.23
192.168.1.103 - - [07/Feb/2026:10:15:37] "POST /api/login HTTP/1.1" 401 67 0.08
192.168.1.101 - - [07/Feb/2026:10:15:38] "GET /api/users HTTP/1.1" 200 1234 0.04
192.168.1.104 - - [07/Feb/2026:10:15:39] "GET /admin HTTP/1.1" 403 89 0.01
192.168.1.100 - - [07/Feb/2026:10:15:40] "GET /api/orders HTTP/1.1" 500 123 6.12
192.168.1.102 - - [07/Feb/2026:10:15:41] "GET /api/products HTTP/1.1" 200 2345 0.15
192.168.1.100 - - [07/Feb/2026:10:15:42] "DELETE /api/users/5 HTTP/1.1" 204 0 0.09
192.168.1.105 - - [07/Feb/2026:10:15:43] "GET /api/users HTTP/1.1" 200 1234 0.06
EOF
```

### **Part 2: Analysis Tasks**

Create `/home/sam/DevopsRoadmap/Phase-1-Linux/text-report.txt` with:

1. **Count total requests**: How many log entries?

2. **Unique IPs**: List all unique IP addresses

3. **Requests per IP**: Count requests from each IP, sorted by count (descending)

4. **HTTP Status Distribution**: Count of each status code (200, 404, 500, etc.)

5. **Find Errors**: Show all lines with status codes 4xx or 5xx

6. **Slowest Requests**: Find requests taking > 1 second

7. **Most Popular Endpoints**: Top 3 most requested paths

8. **Extract Failed Logins**: Find all 401 responses and show their IPs

9. **Calculate Average Response Time**: What's the average response time (last field)?

10. **Transform Data**: Create a CSV with: IP,Status,ResponseTime

### **Security Questions:**

1. Why is `sort` required before `uniq`?
2. What's the difference between `grep "error"` and `grep -w "error"`?
3. Why use `sed -i.bak` instead of `sed -i`?
4. When would you use `awk` instead of `cut`?
5. What does `tail -f` do and why is it critical for production?

---

**Show your commands AND their output for each task!**

---

## 📓 Learning Log

**Date:** February 22, 2026
**Module:** 07 - Text Processing
**Grade:** 87/100

### Mistakes & Corrections

**Mistake 1 — Task 5: `grep` on structured logs can false-match**

Used `grep -E "([45][0-9][0-9])"` which scans the ENTIRE line. If a URL path contains "404" (e.g. `/error-404-page`), a 200 response would be falsely flagged.

```bash
# What I did (risky)
grep -E "([45][0-9][0-9])" access.log

# Correct — target field 8 specifically
awk '$8 ~ /^[45]/' access.log
```

**Rule:** On structured/columnar data (logs, CSVs), use `awk '$FIELD ~ /pattern/'` not raw `grep`.

---

**Mistake 2 — Task 7: `sort -r` is alphabetical, not numeric**

Used `sort -r` which sorts text in reverse alphabetical order. With tied counts, it broke ties alphabetically and silently dropped `/api/login` (also at 2 requests) before `head -3`.

```bash
# What I did (alphabetical, unreliable for numbers)
sort | uniq -c | sort -r | head -3

# Correct — sort -rn for numeric descending
sort | uniq -c | sort -rn | head -3
```

**Rule:** Always use `-n` when sorting counts or any numeric column.

---

**Mistake 3 — Task 10: Commas as separate awk arguments add spaces**

```bash
# What I did — comma separates print arguments, awk uses OFS (space) between them
awk '{print $1,",",$8,",",$10}'   ->   192.168.1.100 , 200 , 0.05

# Correct — embed commas inside the string literal
awk '{print $1","$8","$10}'        ->   192.168.1.100,200,0.05
```

**Rule:** In `awk print`, comma = OFS separator (space by default). To avoid spaces, concatenate with string literals directly, or set `OFS=','`.

---

### Commands to Remember

```bash
# Safe status-code filtering on logs
awk '$8 ~ /^[45]/' access.log

# Always numeric sort on counts
sort | uniq -c | sort -rn | head -N

# Clean CSV from awk
awk '{print $1","$8","$10}' file
# OR set OFS
awk 'BEGIN{OFS=","} {print $1,$8,$10}' file

# Follow logs live in production
tail -F /var/log/app.log | grep --line-buffered "ERROR"
```

### Production Takeaway

Text processing tools are deceptively simple — each has one subtle trap:
- `grep` -> scans whole line, not just a field
- `sort -r` -> alphabetical unless you add `-n`
- `awk print ,` -> adds OFS (space) between args
- `uniq` -> only works on adjacent lines, always sort first

Knowing the traps is what separates a script that works in testing from one that silently lies to you in production at 3 AM.
