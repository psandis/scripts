# Useful Bash Scripts Collection

A curated collection of handy bash one-liners and scripts for everyday system administration and development tasks.

---

## Table of Contents

- [File Operations](#file-operations)
- [Security & Passwords](#security--passwords)
- [Search & Grep](#search--grep)
- [System Monitoring](#system-monitoring)
- [Network & Processes](#network--processes)
- [User Management](#user-management)

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

## Network & Processes

### Find All Ports Used by a Process

List all ports being used by a specific process (by PID).

```bash
netstat -A inet -n -p | grep 1413
```

**Replace `1413` with your process ID.**

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
