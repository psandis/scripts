# Useful Bash Scripts Collection

A curated collection of handy bash one-liners and scripts for everyday system administration and development tasks.

---

## Table of Contents

- [File Operations](#file-operations)
- [Disk Usage & Storage](#disk-usage--storage)
- [Security & Passwords](#security--passwords)
- [Search & Grep](#search--grep)
- [Text Processing](#text-processing)
- [System Monitoring](#system-monitoring)
- [System Information](#system-information)
- [Network & Processes](#network--processes)
- [Archive & Compression](#archive--compression)
- [User Management](#user-management)
- [Git Operations](#git-operations)

---

## File Operations

### Find Unique File Extensions

Find every file and list only the unique file extensions in the current directory tree.

```bash
find . -type f -name "*.*" | awk -F. '{print $NF}' | sort -u
```

**Example output:**
```
js
md
txt
log
```

---

### Empty a Log File

Quickly empty a log or text file without deleting it. ⚠️ **Make sure you have backups!**

```bash
cat /dev/null > /path/to/file.log
```

---

### Find Large Files

Search for files larger than a specific size (example: 500 MB). Useful for identifying files consuming disk space.

```bash
find / -type f -size +500000k -exec ls -lh {} \; | awk '{ print $9 ": " $5 }'
```

**Parameters:**
- `+500000k` = files larger than 500 MB
- Adjust the size value as needed (e.g., `+1000000k` for 1 GB)

---

## Disk Usage & Storage

### Check Directory Size

Display the size of directories in the current location, sorted by size.

```bash
du -sh * | sort -h
```

**Options:**
- `-s` = summary (total for each directory)
- `-h` = human-readable format (KB, MB, GB)
- `sort -h` = sort by human-readable numbers

---

### Top 10 Largest Directories

Find the 10 largest directories from the current location.

```bash
du -h | sort -rh | head -10
```

---

### Check Disk Space Usage

Display disk space usage for all mounted filesystems.

```bash
df -h
```

**Show specific filesystem:**
```bash
df -h /home
```

---

### Find Duplicate Files

Find duplicate files based on MD5 checksum.

```bash
find . -type f -exec md5sum {} \; | sort | uniq -w32 -dD
```

---

### Monitor Disk I/O in Real-Time

Watch disk I/O statistics in real-time.

```bash
iostat -x 2
```

**Parameters:**
- `2` = refresh every 2 seconds
- `-x` = extended statistics

---

### Check Inode Usage

Check inode usage (useful when disk shows space but can't create files).

```bash
df -i
```

---

## Security & Passwords

### Generate Random Password

Generate a random password with a specified length (default: 20 characters). Uses uppercase, lowercase letters, numbers, and underscores.

**Add to `~/.bashrc`:**

```bash
genpasswrd() {
    local l=$1
    [ "$l" == "" ] && l=20
    tr -dc A-Za-z0-9_ < /dev/urandom | head -c ${l} | xargs
}
```

**Usage:**

```bash
# Generate 16-character password
genpasswrd 16

# Generate default 20-character password
genpasswrd
```

**Standalone version (without function):**

```bash
# Replace ${l} with desired length
tr -dc A-Za-z0-9_ < /dev/urandom | head -c 20 | xargs
```

---

### MD5 Encrypt Password

Generate an MD5 hash of a plaintext password.

```bash
echo -n "your_plaintext_password" | md5sum -
```

> **Note:** MD5 is not recommended for secure password storage. Consider using bcrypt, scrypt, or Argon2 for production systems.

---

### SHA256 Hash Generation

Generate a SHA256 hash (more secure than MD5).

```bash
echo -n "your_plaintext_password" | sha256sum
```

---

### Generate SSH Key Pair

Generate a new SSH key pair for secure authentication.

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

**For RSA (4096-bit):**
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

---

### Check File Permissions Recursively

Find files with specific permissions (e.g., world-writable files).

```bash
find . -type f -perm 0777
```

**Find and fix world-writable files:**
```bash
find . -type f -perm 0777 -exec chmod 644 {} \;
```

---

## Search & Grep

### Search String in JavaScript Files

Search for a specific string in all `.js` files within the current directory tree.

```bash
find . -name '*.js' -exec grep -i 'string to search for' {} \; -print
```

**Options:**
- `-i` = case-insensitive search
- Remove `-i` for case-sensitive search
- Replace `*.js` with other extensions as needed (e.g., `*.py`, `*.txt`)

---

### Grep Entries by Date and Time Range

Extract log entries for a specific day within a time range.

```bash
less [FILE] | grep -E '2020.01.16 (09:[0-5][0-9]|10:00)'
```

**Example:** This searches for entries on `2020.01.16` between `09:00` and `10:00`.

**Customize:**
- Change date: `2020.01.16` → your date
- Change time range: `(09:[0-5][0-9]|10:00)` → your range

---

## Text Processing

### Count Lines, Words, and Characters

Count lines, words, and characters in a file.

```bash
wc file.txt
```

**Count only lines:**
```bash
wc -l file.txt
```

---

### Remove Duplicate Lines

Remove duplicate lines from a file while preserving order.

```bash
awk '!seen[$0]++' file.txt
```

**Using sort and uniq:**
```bash
sort file.txt | uniq
```

---

### Extract Specific Columns

Extract specific columns from a file (e.g., columns 1 and 3).

```bash
awk '{print $1, $3}' file.txt
```

**Using cut for delimiter-separated files:**
```bash
cut -d',' -f1,3 file.csv
```

---

### Find and Replace Text in Files

Replace text in a file (in-place).

```bash
sed -i 's/old_text/new_text/g' file.txt
```

**macOS version (requires backup extension):**
```bash
sed -i '.bak' 's/old_text/new_text/g' file.txt
```

---

### Convert File to Lowercase/Uppercase

Convert entire file to lowercase.

```bash
tr '[:upper:]' '[:lower:]' < input.txt > output.txt
```

**Convert to uppercase:**
```bash
tr '[:lower:]' '[:upper:]' < input.txt > output.txt
```

---

### Merge Lines from Multiple Files

Paste lines from multiple files side by side.

```bash
paste file1.txt file2.txt
```

---

### Print Specific Line Number

Print line 42 from a file.

```bash
sed -n '42p' file.txt
```

**Print lines 10-20:**
```bash
sed -n '10,20p' file.txt
```

---

## System Monitoring

### Check Running Processes on Specific Port

Check which processes are using a specific port (example: FTP port 21).

```bash
lsof -ni:21
```

**Replace `21` with any port number:**
- `80` for HTTP
- `443` for HTTPS
- `3306` for MySQL
- `5432` for PostgreSQL

---

### Display Active Service Port

Show active (LISTEN) ports for a specific service (example: Apache/HTTPD).

```bash
netstat -tulpen | grep "HTTPD"
```

**Options:**
- `-t` = TCP connections
- `-u` = UDP connections
- `-l` = listening sockets
- `-p` = show PID/program name
- `-e` = extended information
- `-n` = numeric addresses

---

### Find Application by PID

Display the application/source assigned to a specific process ID.

```bash
ps aux | grep "HTTPD"
```

---

### Monitor Memory Usage

Display memory usage in real-time.

```bash
watch -n 1 free -m
```

---

### Top Processes by CPU Usage

Display top processes sorted by CPU usage.

```bash
ps aux --sort=-%cpu | head -10
```

---

### Top Processes by Memory Usage

Display top processes sorted by memory usage.

```bash
ps aux --sort=-%mem | head -10
```

---

### Monitor System Resources with htop

Interactive process viewer (if installed).

```bash
htop
```

**Install htop:**
```bash
# Ubuntu/Debian
sudo apt install htop

# macOS
brew install htop
```

---

## System Information

### Check OS Version

Display operating system information.

```bash
# Linux
cat /etc/os-release

# macOS
sw_vers

# Universal
uname -a
```

---

### Check CPU Information

Display CPU details.

```bash
# Linux
lscpu

# macOS
sysctl -n machdep.cpu.brand_string

# Alternative
cat /proc/cpuinfo
```

---

### Check System Uptime

Display how long the system has been running.

```bash
uptime
```

---

### Check Kernel Version

Display kernel version.

```bash
uname -r
```

---

### List Hardware Information

Display detailed hardware information (Linux).

```bash
lshw -short
```

---

### Check Memory Information

Display detailed memory information.

```bash
# Linux
cat /proc/meminfo

# macOS
vm_stat
```

---

## Network & Processes

### Find All Ports Used by a Process

List all ports being used by a specific process (by PID).

```bash
netstat -A inet -n -p | grep 1413
```

**Replace `1413` with your process ID.**

---

### Check Network Connectivity

Test network connectivity to a host.

```bash
ping -c 4 google.com
```

**Parameters:**
- `-c 4` = send 4 packets (omit for continuous ping)

---

### Trace Network Route

Display the route packets take to reach a host.

```bash
traceroute google.com

# macOS alternative
traceroute google.com
```

---

### Check Open Network Connections

Display all open network connections.

```bash
netstat -an
```

---

### Download File from URL

Download a file using curl or wget.

```bash
# Using curl
curl -O https://example.com/file.zip

# Using wget
wget https://example.com/file.zip
```

---

### Check Public IP Address

Display your public IP address.

```bash
curl ifconfig.me
```

**Alternatives:**
```bash
curl ipinfo.io/ip
curl icanhazip.com
```

---

### Test Port Connectivity

Test if a specific port is open on a remote host.

```bash
nc -zv example.com 80
```

**Using telnet:**
```bash
telnet example.com 80
```

---

### Display Network Interface Information

Show network interface configuration.

```bash
# Linux
ip addr show

# macOS/BSD
ifconfig
```

---

## Archive & Compression

### Create tar.gz Archive

Create a compressed archive of a directory.

```bash
tar -czf archive.tar.gz /path/to/directory
```

**Options:**
- `-c` = create archive
- `-z` = compress with gzip
- `-f` = specify filename

---

### Extract tar.gz Archive

Extract a compressed archive.

```bash
tar -xzf archive.tar.gz
```

**Extract to specific directory:**
```bash
tar -xzf archive.tar.gz -C /path/to/destination
```

---

### Create zip Archive

Create a zip archive.

```bash
zip -r archive.zip /path/to/directory
```

---

### Extract zip Archive

Extract a zip archive.

```bash
unzip archive.zip
```

**Extract to specific directory:**
```bash
unzip archive.zip -d /path/to/destination
```

---

### List Archive Contents

List contents without extracting.

```bash
# tar.gz
tar -tzf archive.tar.gz

# zip
unzip -l archive.zip
```

---

### Compress Single File with gzip

Compress a single file.

```bash
gzip file.txt
```

**Decompress:**
```bash
gunzip file.txt.gz
```

---

### Create tar.bz2 Archive

Create a bzip2 compressed archive (better compression than gzip).

```bash
tar -cjf archive.tar.bz2 /path/to/directory
```

**Extract:**
```bash
tar -xjf archive.tar.bz2
```

---

## User Management

### List Currently Logged In Users

Get a unique list of all usernames currently logged into the system.

```bash
who | cut -d' ' -f1 | sort | uniq
```

**Breakdown:**
- `who` = show logged-in users
- `cut -d' ' -f1` = extract first field (username)
- `sort` = alphabetically sort
- `uniq` = remove duplicates

---

### Switch User

Switch to another user account.

```bash
su - username
```

---

### Add New User

Create a new user account (requires root/sudo).

```bash
# Linux
sudo useradd -m -s /bin/bash username

# macOS
sudo dscl . -create /Users/username
```

---

### Change User Password

Change password for a user.

```bash
# Change your own password
passwd

# Change another user's password (requires sudo)
sudo passwd username
```

---

### List All Users

List all users on the system.

```bash
# Linux
cat /etc/passwd | cut -d: -f1

# macOS
dscl . list /Users
```

---

### Check User Groups

Display groups a user belongs to.

```bash
groups username
```

---

### Last Login Information

Display last login information for users.

```bash
last
```

---

## Git Operations

### View Commit History

Display commit history with one line per commit.

```bash
git log --oneline --graph --all
```

---

### Undo Last Commit (Keep Changes)

Undo the last commit but keep the changes staged.

```bash
git reset --soft HEAD~1
```

---

### Discard All Local Changes

Discard all local changes and reset to last commit.

```bash
git reset --hard HEAD
```

---

### Delete Local Branches Not on Remote

Clean up local branches that have been deleted on remote.

```bash
git fetch --prune
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -D
```

---

### View File Changes in Last Commit

Show what changed in the last commit.

```bash
git show HEAD
```

---

### Search Commit Messages

Search for commits containing specific text.

```bash
git log --all --grep="search term"
```

---

### Show File History

Display all commits that modified a specific file.

```bash
git log --follow -- path/to/file
```

---

### Create and Switch to New Branch

Create a new branch and switch to it.

```bash
git checkout -b new-branch-name
```

---

### Stash Changes

Temporarily save uncommitted changes.

```bash
# Stash changes
git stash

# List stashes
git stash list

# Apply most recent stash
git stash pop
```

---

### View Differences

Show unstaged changes.

```bash
git diff
```

**Show staged changes:**
```bash
git diff --staged
```

---

### Remove Untracked Files

Remove untracked files and directories.

```bash
# Dry run (see what would be deleted)
git clean -n

# Actually delete
git clean -fd
```

---

## Contributing

Feel free to submit pull requests with additional useful scripts! Please follow the existing format:
- Clear description
- Working code example
- Usage instructions
- Any relevant notes or warnings

---

## License

This collection is provided as-is for educational and practical purposes. Use at your own risk.

---

## Disclaimer

⚠️ **Always test scripts in a safe environment before running them on production systems.**

Some commands (especially those involving `find /`, `rm`, or file modifications) can have significant system impact. Always:
- Have backups
- Understand what the command does
- Test in a non-production environment first
- Use appropriate permissions
