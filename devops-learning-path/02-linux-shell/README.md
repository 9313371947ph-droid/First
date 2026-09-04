# Module 02: Linux & Shell Scripting for DevOps

## 🎯 Learning Objectives

By the end of this module, you will:
- Master essential Linux commands for DevOps
- Understand Linux file system and permissions
- Write practical bash scripts for automation
- Process and manipulate text data efficiently
- Debug and optimize shell scripts

---

## 📚 Part 1: Why Linux is Critical for DevOps

### The Reality

- **90%+** of cloud workloads run on Linux
- **All** major container technologies are Linux-native
- **Most** DevOps tools are built for Linux first
- **Servers** in production are predominantly Linux-based

### Key Reasons

1. **Stability**: Linux servers can run for years without reboot
2. **Performance**: Efficient resource utilization
3. **Security**: Robust permission model
4. **Automation**: Built for scripting and automation
5. **Cost**: Open source and free
6. **Community**: Massive ecosystem and support

---

## 🔧 Part 2: Essential Linux Commands

### File System Navigation

```bash
# Where am I?
pwd

# List files and directories
ls                    # Basic listing
ls -la                # Detailed view with hidden files
ls -lh                # Human-readable sizes
tree                  # Directory tree structure (install if needed)

# Change directory
cd /path/to/dir       # Absolute path
cd ..                 # Go up one level
cd ~                  # Go to home directory
cd -                  # Go to previous directory

# Create directories
mkdir new_directory
mkdir -p parent/child/grandchild    # Create nested directories
```

### File Operations

```bash
# Copy files/directories
cp file1.txt file2.txt
cp -r dir1/ dir2/                   # Recursive copy

# Move/Rename
mv old_name.txt new_name.txt
mv file.txt /path/to/destination/

# Remove (CAREFUL!)
rm file.txt
rm -r directory/                    # Recursive removal
rm -rf directory/                   # Force removal (DANGEROUS!)

# View file contents
cat file.txt                        # Display entire file
less file.txt                       # Paginated view
head -n 20 file.txt                 # First 20 lines
tail -n 20 file.txt                 # Last 20 lines
tail -f logfile.log                 # Follow log file in real-time
```

### File Permissions

```bash
# View permissions
ls -la

# Understanding permissions: -rwxrwxrwx
# First character: file type (- = file, d = directory)
# Next 9 characters: owner, group, others (read, write, execute)

# Change permissions (numeric mode)
chmod 755 script.sh                 # rwxr-xr-x
chmod 644 file.txt                  # rw-r--r--
chmod 600 secret.key                # rw-------

# Change permissions (symbolic mode)
chmod +x script.sh                  # Add execute permission
chmod u+x script.sh                 # Add execute for user
chmod go-w file.txt                 # Remove write for group and others

# Change ownership
chown user:group file.txt
chown -R user:group directory/      # Recursive
```

### Process Management

```bash
# View running processes
ps aux                              # All processes
top                                 # Dynamic process view
htop                                # Enhanced top (install if needed)

# Find processes
ps aux | grep nginx
pgrep nginx

# Kill processes
kill PID                            # Graceful termination
kill -9 PID                         # Force kill (use carefully!)
pkill process_name
killall nginx

# Background/Foreground jobs
./long_script.sh &                  # Run in background
jobs                                # List background jobs
fg %1                               # Bring job 1 to foreground
Ctrl+Z                              # Suspend current job
bg                                  # Resume suspended job in background
```

### System Information

```bash
# System info
uname -a                            # Kernel and system info
hostname                            # System hostname
uptime                              # System uptime and load

# Resource usage
free -h                             # Memory usage
df -h                               # Disk usage
du -sh directory/                   # Directory size

# CPU info
lscpu                               # CPU details
nproc                               # Number of processors

# Network info
ip addr                             # IP addresses
ip route                            # Routing table
ss -tulpn                           # Listening ports
ping google.com                     # Test connectivity
curl -I https://example.com         # HTTP request
```

### Text Processing (CRITICAL for DevOps!)

```bash
# Search text
grep "error" logfile.log
grep -i "error" logfile.log         # Case insensitive
grep -r "pattern" /path/            # Recursive search
grep -v "info" logfile.log          # Inverse match (exclude)
grep -c "error" logfile.log         # Count matches

# Find files
find /path -name "*.log"
find /path -type f -size +100M      # Files larger than 100MB
find /path -mtime -7                # Modified in last 7 days

# Stream editor
sed 's/old/new/g' file.txt          # Replace all occurrences
sed -i 's/old/new/g' file.txt       # Edit file in-place
sed -n '10,20p' file.txt            # Print lines 10-20

# Column extractor
cut -d: -f1 /etc/passwd             # First field, colon delimiter
cut -d' ' -f3 access.log            # Third field, space delimiter

# Sort and unique
sort file.txt                       # Sort lines
sort -n file.txt                    # Numeric sort
uniq file.txt                       # Remove consecutive duplicates
sort file.txt | uniq                # Sort then remove duplicates
sort | uniq -c                      # Count occurrences
sort | uniq -c | sort -rn           # Count and sort by frequency

# Word count
wc file.txt                         # Lines, words, bytes
wc -l file.txt                      # Line count only
```

### Compression & Archiving

```bash
# tar (Tape Archive)
tar -cvf archive.tar file1 file2    # Create archive
tar -xvf archive.tar                # Extract archive
tar -czvf archive.tar.gz dir/       # Create gzipped archive
tar -xzvf archive.tar.gz            # Extract gzipped archive
tar -tzf archive.tar.gz             # List contents

# zip/unzip
zip -r archive.zip directory/
unzip archive.zip

# gzip
gzip file.txt                       # Compress (creates file.txt.gz)
gunzip file.txt.gz                  # Decompress
```

---

## 📝 Part 3: Bash Scripting Fundamentals

### Your First Script

```bash
#!/bin/bash
# This is a shebang - tells system to use bash interpreter

# Comments start with #
echo "Hello, DevOps Engineer!"

# Variables (no $ when assigning)
NAME="DevOps Learner"
echo "Welcome, $NAME!"

# Command substitution
CURRENT_DATE=$(date)
echo "Today is: $CURRENT_DATE"
```

### Making Scripts Executable

```bash
chmod +x my_script.sh
./my_script.sh
```

### Variables

```bash
#!/bin/bash

# String variables
MESSAGE="Hello World"
echo $MESSAGE
echo "${MESSAGE}"                     # Safer syntax

# Read-only variables
readonly PI=3.14159

# Reading user input
echo "What's your name?"
read NAME
echo "Hello, $NAME"

# Command line arguments
echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "All arguments: $@"
echo "Number of arguments: $#"

# Special variables
echo "Process ID: $$"
echo "Last exit code: $?"
```

### Conditional Statements

```bash
#!/bin/bash

# If statement
AGE=25

if [ $AGE -lt 18 ]; then
    echo "You are a minor"
elif [ $AGE -lt 65 ]; then
    echo "You are an adult"
else
    echo "You are a senior"
fi

# Comparison operators
# Numbers: -eq, -ne, -gt, -lt, -ge, -le
# Strings: =, !=, -z (empty), -n (not empty)
# Files: -e (exists), -f (file), -d (directory), -r (readable), -w (writable), -x (executable)

# File check example
FILE="/etc/passwd"

if [ -f "$FILE" ]; then
    echo "$FILE exists and is a file"
else
    echo "$FILE does not exist"
fi

# Case statement
DAY="Monday"

case $DAY in
    Monday|Tuesday|Wednesday|Thursday|Friday)
        echo "It's a weekday"
        ;;
    Saturday|Sunday)
        echo "It's the weekend!"
        ;;
    *)
        echo "Invalid day"
        ;;
esac
```

### Loops

```bash
#!/bin/bash

# For loop
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

# For loop with range
for i in {1..5}; do
    echo "Number: $i"
done

# For loop over files
for file in *.txt; do
    echo "Processing $file"
done

# While loop
COUNT=1
while [ $COUNT -le 5 ]; do
    echo "Count: $COUNT"
    ((COUNT++))
done

# Until loop (runs until condition is true)
UNTIL_COUNT=5
until [ $UNTIL_COUNT -eq 0 ]; do
    echo "Countdown: $UNTIL_COUNT"
    ((UNTIL_COUNT--))
done

# Break and continue
for i in {1..10}; do
    if [ $i -eq 5 ]; then
        continue                    # Skip iteration
    fi
    if [ $i -eq 8 ]; then
        break                       # Exit loop
    fi
    echo $i
done
```

### Functions

```bash
#!/bin/bash

# Define function
greet() {
    echo "Hello, $1!"
}

# Call function
greet "DevOps Engineer"

# Function with return value
add() {
    local sum=$(($1 + $2))
    echo $sum
}

RESULT=$(add 5 3)
echo "5 + 3 = $RESULT"

# Function with multiple returns
get_stats() {
    local count=$(wc -l < "$1")
    local words=$(wc -w < "$1")
    echo "$count $words"
}

read LINES WORDS <<< $(get_stats myfile.txt)
echo "Lines: $LINES, Words: $WORDS"
```

---

## 🛠️ Part 4: Practical DevOps Scripts

### Script 1: Health Check Automation

```bash
#!/bin/bash
# health_check.sh - System health monitoring

LOG_FILE="/var/log/health_check.log"
THRESHOLD_DISK=80
THRESHOLD_MEMORY=90

log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

check_disk_usage() {
    log_message "Checking disk usage..."
    USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')
    
    if [ $USAGE -ge $THRESHOLD_DISK ]; then
        log_message "WARNING: Disk usage is ${USAGE}%"
        return 1
    else
        log_message "OK: Disk usage is ${USAGE}%"
        return 0
    fi
}

check_memory_usage() {
    log_message "Checking memory usage..."
    USAGE=$(free | awk 'NR==2 {printf "%.0f", $3*100/$2}')
    
    if [ $USAGE -ge $THRESHOLD_MEMORY ]; then
        log_message "WARNING: Memory usage is ${USAGE}%"
        return 1
    else
        log_message "OK: Memory usage is ${USAGE}%"
        return 0
    fi
}

check_service() {
    SERVICE=$1
    log_message "Checking service: $SERVICE"
    
    if systemctl is-active --quiet $SERVICE; then
        log_message "OK: $SERVICE is running"
        return 0
    else
        log_message "WARNING: $SERVICE is not running"
        return 1
    fi
}

# Main execution
log_message "========== Starting Health Check =========="

check_disk_usage
DISK_STATUS=$?

check_memory_usage
MEMORY_STATUS=$?

check_service "nginx"
NGINX_STATUS=$?

check_service "docker"
DOCKER_STATUS=$?

log_message "========== Health Check Complete =========="

# Exit with error if any check failed
if [ $DISK_STATUS -ne 0 ] || [ $MEMORY_STATUS -ne 0 ] || \
   [ $NGINX_STATUS -ne 0 ] || [ $DOCKER_STATUS -ne 0 ]; then
    log_message "RESULT: Some checks failed!"
    exit 1
else
    log_message "RESULT: All checks passed!"
    exit 0
fi
```

### Script 2: Log Analyzer

```bash
#!/bin/bash
# log_analyzer.sh - Analyze web server logs

LOG_FILE="${1:-/var/log/nginx/access.log}"
OUTPUT_DIR="./log_analysis"

if [ ! -f "$LOG_FILE" ]; then
    echo "Error: Log file not found: $LOG_FILE"
    exit 1
fi

mkdir -p $OUTPUT_DIR

echo "Analyzing log file: $LOG_FILE"
echo "=========================="

# Top 10 IP addresses
echo -e "\n📊 Top 10 IP Addresses:"
awk '{print $1}' $LOG_FILE | sort | uniq -c | sort -rn | head -10

# Top 10 requested URLs
echo -e "\n🔗 Top 10 Requested URLs:"
awk '{print $7}' $LOG_FILE | sort | uniq -c | sort -rn | head -10

# HTTP Status codes distribution
echo -e "\n📈 HTTP Status Codes:"
awk '{print $9}' $LOG_FILE | sort | uniq -c | sort -rn

# Error requests (4xx and 5xx)
echo -e "\n❌ Error Requests:"
ERRORS=$(awk '$9 ~ /^[45]/' $LOG_FILE | wc -l)
TOTAL=$(wc -l < $LOG_FILE)
ERROR_RATE=$(echo "scale=2; $ERRORS * 100 / $TOTAL" | bc)
echo "Total errors: $ERRORS out of $TOTAL requests (${ERROR_RATE}%)"

# Requests per hour
echo -e "\n⏰ Requests per Hour:"
awk -F: '{print $2}' $LOG_FILE | sort | uniq -c | sort -k2n

# Save report
REPORT_FILE="$OUTPUT_DIR/report_$(date +%Y%m%d_%H%M%S).txt"
{
    echo "Log Analysis Report"
    echo "Generated: $(date)"
    echo "Log file: $LOG_FILE"
    echo ""
    echo "Top 10 IPs:"
    awk '{print $1}' $LOG_FILE | sort | uniq -c | sort -rn | head -10
} > $REPORT_FILE

echo -e "\n✅ Report saved to: $REPORT_FILE"
```

### Script 3: Automated Backup

```bash
#!/bin/bash
# backup.sh - Automated backup script

BACKUP_DIR="/backups"
SOURCE_DIRS=("/home" "/etc" "/var/www")
RETENTION_DAYS=7
DATE_STAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${DATE_STAMP}.tar.gz"

# Create backup directory if it doesn't exist
mkdir -p $BACKUP_DIR

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

log "Starting backup..."

# Create compressed archive
log "Creating backup archive: $BACKUP_NAME"
tar -czf "${BACKUP_DIR}/${BACKUP_NAME}" "${SOURCE_DIRS[@]}" 2>/dev/null

if [ $? -eq 0 ]; then
    log "Backup created successfully"
    
    # Get backup size
    BACKUP_SIZE=$(du -h "${BACKUP_DIR}/${BACKUP_NAME}" | cut -f1)
    log "Backup size: $BACKUP_SIZE"
    
    # Cleanup old backups
    log "Cleaning up backups older than $RETENTION_DAYS days..."
    find $BACKUP_DIR -name "backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete
    
    log "Old backups removed"
else
    log "ERROR: Backup failed!"
    exit 1
fi

log "Backup completed successfully"
```

### Script 4: Docker Container Monitor

```bash
#!/bin/bash
# docker_monitor.sh - Monitor Docker containers

ALERT_EMAIL="admin@example.com"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

check_containers() {
    log "Checking Docker containers..."
    
    # Get all container names
    CONTAINERS=$(docker ps -a --format '{{.Names}}')
    
    for CONTAINER in $CONTAINERS; do
        STATUS=$(docker inspect -f '{{.State.Status}}' $CONTAINER)
        
        if [ "$STATUS" != "running" ]; then
            log "WARNING: Container $CONTAINER is $STATUS"
            
            # Try to restart
            log "Attempting to restart $CONTAINER..."
            docker start $CONTAINER
            
            if [ $? -eq 0 ]; then
                log "Container $CONTAINER restarted successfully"
            else
                log "ERROR: Failed to restart $CONTAINER"
                # Send alert (implement email/slack notification)
                # mail -s "Container Alert: $CONTAINER" $ALERT_EMAIL
            fi
        fi
    done
}

check_disk_space() {
    log "Checking Docker disk usage..."
    
    # Docker system-wide disk usage
    docker system df
    
    # Prune if necessary
    USAGE=$(docker system df --format '{{.Size}}' | grep -o '[0-9]*' | head -1)
    
    if [ $USAGE -gt 10 ]; then  # If more than 10GB
        log "High disk usage detected, pruning..."
        docker system prune -f
    fi
}

check_logs() {
    CONTAINER=$1
    PATTERN=$2
    
    log "Checking logs for $CONTAINER with pattern: $PATTERN"
    docker logs --tail 100 $CONTAINER 2>&1 | grep -i "$PATTERN"
}

# Main execution
log "========== Docker Monitor Started =========="

check_containers
check_disk_space

# Check specific containers for errors
for CONTAINER in $(docker ps --format '{{.Names}}'); do
    check_logs $CONTAINER "error"
    check_logs $CONTAINER "exception"
    check_logs $CONTAINER "failed"
done

log "========== Docker Monitor Complete =========="
```

---

## 🏋️ Practical Exercises

### Exercise 1: Linux Command Mastery (2 hours)

**Task**: Complete these operations on your system:

1. Navigate to `/var/log` and list all `.log` files sorted by size
2. Find all files modified in the last 24 hours
3. Count how many processes are running as root
4. Extract all unique IP addresses from a log file
5. Create a directory structure: `test/{dir1,dir2,dir3}/{sub1,sub2}`
6. Set permissions so only owner can read/write/execute a script

**Deliverable**: Document commands used and their output

### Exercise 2: Build a System Info Script (1 hour)

**Task**: Create a script that displays:
- Hostname and OS version
- CPU count and memory total
- Disk usage for root partition
- List of running services
- Uptime and load average

**Requirements**:
- Use functions for each section
- Format output nicely with colors
- Save output to a file with timestamp

### Exercise 3: Log Parser Challenge (2 hours)

**Task**: Create a script that parses Apache/Nginx access logs and reports:
- Total requests
- Unique visitors (IPs)
- Most visited pages
- Error rate (4xx and 5xx responses)
- Average response size
- Requests per hour

**Bonus**: Generate an HTML report

### Exercise 4: Deployment Helper Script (2 hours)

**Task**: Create a deployment script that:
1. Pulls latest code from git
2. Runs tests (simulate with echo)
3. Creates a backup of current version
4. Deploys new version
5. Restarts services
6. Validates deployment
7. Rolls back if validation fails

**Requirements**:
- Use proper error handling
- Log all actions
- Support rollback functionality

### Exercise 5: User Management Script (1 hour)

**Task**: Create a script to manage system users:
- Add new user with home directory
- Set password
- Add to specific groups
- Create SSH key structure
- Set proper permissions

**Input**: Username and group as command-line arguments

---

## 💡 Best Practices for Shell Scripting

### 1. Always Use Shebang
```bash
#!/bin/bash
```

### 2. Enable Strict Mode
```bash
set -euo pipefail
# -e: Exit on error
# -u: Error on undefined variables
# -o pipefail: Pipeline fails if any command fails
```

### 3. Quote Your Variables
```bash
# Good
echo "$VARIABLE"

# Bad (can break with spaces)
echo $VARIABLE
```

### 4. Validate Inputs
```bash
if [ -z "$1" ]; then
    echo "Error: Argument required"
    exit 1
fi
```

### 5. Use Functions
```bash
# Organize code into reusable functions
main() {
    validate_inputs
    setup_environment
    execute_task
    cleanup
}
```

### 6. Proper Error Handling
```bash
if ! command; then
    echo "Error: Command failed"
    exit 1
fi

# Or
command || {
    echo "Error: Command failed"
    exit 1
}
```

### 7. Comment Your Code
```bash
# WHY, not just WHAT
# Bad: Increment counter
# Good: Move to next batch for processing
((COUNTER++))
```

### 8. Test Thoroughly
- Test with different inputs
- Test edge cases
- Test error conditions
- Use shellcheck for linting

---

## 🔍 Debugging Techniques

### Enable Debug Mode
```bash
bash -x script.sh
```

### Debug Within Script
```bash
set -x          # Enable debug
# ... code ...
set +x          # Disable debug
```

### Print Variables
```bash
echo "DEBUG: VARIABLE=$VARIABLE" >&2
```

### Use set -e for Early Exit
```bash
set -e          # Exit immediately on error
```

### Install ShellCheck
```bash
# Ubuntu/Debian
sudo apt install shellcheck

# macOS
brew install shellcheck

# Run
shellcheck my_script.sh
```

---

## 📝 Knowledge Check

### Quiz Questions

1. **What does `chmod 755` mean?**
   <details>
   <summary>Click for Answer</summary>
   
   Owner: rwx (read, write, execute)
   Group: r-x (read, execute)
   Others: r-x (read, execute)
   </details>

2. **How do you find all files larger than 100MB?**
   <details>
   <summary>Click for Answer</summary>
   
   ```bash
   find /path -type f -size +100M
   ```
   </details>

3. **What's the difference between `$@` and `$*`?**
   <details>
   <summary>Click for Answer</summary>
   
   - `$@`: Treats each argument as separate (better for loops)
   - `$*`: Treats all arguments as single string
   </details>

4. **How do you check if a file exists in bash?**
   <details>
   <summary>Click for Answer</summary>
   
   ```bash
   if [ -f "filename" ]; then
       echo "File exists"
   fi
   ```
   </details>

5. **What does `set -euo pipefail` do?**
   <details>
   <summary>Click for Answer</summary>
   
   - `-e`: Exit on error
   - `-u`: Error on undefined variables
   - `-o pipefail`: Pipeline fails if any command fails
   </details>

---

## 📚 Additional Resources

### Books
- "Linux Command Line and Shell Scripting Bible" by Richard Blum
- "Classic Shell Scripting" by Arnold Robbins
- "The Linux Command Line" by William Shotts (Free online!)

### Online Resources
- [ExplainShell](https://explainshell.com/) - Understand complex commands
- [ShellCheck](https://www.shellcheck.net/) - Script linting
- [GNU Bash Manual](https://www.gnu.org/software/bash/manual/)

### Practice Platforms
- [OverTheWire Bandit](https://overthewire.org/wargames/bandit/) - Linux security wargame
- [CmdChallenge](https://cmdchallenge.com/) - Command line challenges

### Cheat Sheets
- [Linux Command Library](https://linuxcommandlibrary.com/)
- [DevOps Cheat Sheet](https://cheatsheet.dennyzhang.com/)

---

## 🎯 Next Steps

✅ Complete all practical exercises
✅ Write at least 5 useful scripts for your workflow
✅ Install and use ShellCheck on all scripts
✅ Practice daily with Linux command line
✅ Proceed to [Module 03: Version Control with Git](../03-version-control-git/README.md)

---

## 💬 Reflection Questions

1. Which Linux commands do you use most frequently?
2. What tasks in your workflow could be automated with scripts?
3. How has learning shell scripting changed your approach to problem-solving?
4. What's the most complex script you can imagine building?

---

*"Automation is the heart of DevOps, and shell scripting is the foundation of automation."*
