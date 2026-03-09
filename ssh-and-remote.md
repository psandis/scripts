# SSH & Remote Access Scripts

A curated collection of SSH, SCP, rsync, and remote access commands for secure server management.

---

## Table of Contents

- [SSH Basics](#ssh-basics)
- [SSH Keys](#ssh-keys)
- [SSH Config](#ssh-config)
- [Port Forwarding & Tunneling](#port-forwarding--tunneling)
- [SCP — Secure Copy](#scp--secure-copy)
- [rsync](#rsync)
- [Remote Commands](#remote-commands)
- [SSH Agent](#ssh-agent)
- [Troubleshooting](#troubleshooting)

---

## SSH Basics

### Connect to a Server

```bash
ssh user@hostname
```

**With a specific port:**
```bash
ssh -p 2222 user@hostname
```

**With a specific key:**
```bash
ssh -i ~/.ssh/my_key user@hostname
```

---

### Disconnect

```bash
exit
# or press Ctrl+D
```

---

### Keep Connection Alive

Send keepalive packets every 60 seconds:

```bash
ssh -o ServerAliveInterval=60 user@hostname
```

---

### Connect with Verbose Output (Debugging)

```bash
ssh -v user@hostname
# -vv or -vvv for more verbosity
```

---

## SSH Keys

### Generate a New SSH Key Pair

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

**RSA (4096-bit) for older systems:**
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

---

### Copy Public Key to Server

```bash
ssh-copy-id user@hostname
```

**With a specific key:**
```bash
ssh-copy-id -i ~/.ssh/my_key.pub user@hostname
```

**Manual alternative:**
```bash
cat ~/.ssh/id_ed25519.pub | ssh user@hostname "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

---

### List Your SSH Keys

```bash
ls -la ~/.ssh
```

---

### View Public Key

```bash
cat ~/.ssh/id_ed25519.pub
```

---

### Change Key Passphrase

```bash
ssh-keygen -p -f ~/.ssh/id_ed25519
```

---

### Get Key Fingerprint

```bash
ssh-keygen -lf ~/.ssh/id_ed25519.pub
```

---

## SSH Config

The `~/.ssh/config` file lets you define shortcuts and defaults per host.

### Example Config

```
# Default settings for all hosts
Host *
  ServerAliveInterval 60
  AddKeysToAgent yes

# Production server
Host prod
  HostName 192.168.1.100
  User deploy
  IdentityFile ~/.ssh/prod_key
  Port 2222

# Jump through a bastion host
Host internal
  HostName 10.0.0.5
  User admin
  ProxyJump bastion

Host bastion
  HostName bastion.example.com
  User ec2-user
  IdentityFile ~/.ssh/bastion_key
```

**Usage:**
```bash
ssh prod
ssh internal
```

---

### Set Correct Permissions on Config

```bash
chmod 600 ~/.ssh/config
chmod 700 ~/.ssh
```

---

## Port Forwarding & Tunneling

### Local Port Forwarding

Forward local port 8080 to remote port 80:

```bash
ssh -L 8080:localhost:80 user@hostname
```

Access a remote database locally:
```bash
ssh -L 5432:localhost:5432 user@db-server
# Now connect to localhost:5432
```

---

### Remote Port Forwarding

Expose local port 3000 on the remote server's port 9090:

```bash
ssh -R 9090:localhost:3000 user@hostname
```

---

### Dynamic Port Forwarding (SOCKS Proxy)

Route traffic through a remote server:

```bash
ssh -D 1080 user@hostname
# Configure browser to use SOCKS5 proxy at localhost:1080
```

---

### SSH Tunnel in Background

```bash
ssh -fNL 5432:localhost:5432 user@db-server
```

**Options:**
- `-f` = go to background
- `-N` = don't execute remote command

**Kill the tunnel:**
```bash
pkill -f "ssh -fNL"
```

---

### Jump Host (Bastion)

Connect to an internal server via a bastion:

```bash
ssh -J bastion-user@bastion.example.com internal-user@10.0.0.5
```

---

## SCP — Secure Copy

### Copy File to Remote Server

```bash
scp file.txt user@hostname:/remote/path/
```

---

### Copy File from Remote Server

```bash
scp user@hostname:/remote/path/file.txt ./
```

---

### Copy Directory Recursively

```bash
scp -r ./local-dir user@hostname:/remote/path/
```

---

### Copy with a Specific Port

```bash
scp -P 2222 file.txt user@hostname:/remote/path/
```

---

### Copy with a Specific Key

```bash
scp -i ~/.ssh/my_key file.txt user@hostname:/remote/path/
```

---

## rsync

rsync is preferred over SCP for large transfers — it's faster, resumable, and supports incremental syncs.

### Sync Local Directory to Remote

```bash
rsync -avz ./local-dir/ user@hostname:/remote/path/
```

**Options:**
- `-a` = archive mode (preserves permissions, timestamps, symlinks)
- `-v` = verbose
- `-z` = compress during transfer

---

### Sync Remote Directory to Local

```bash
rsync -avz user@hostname:/remote/path/ ./local-dir/
```

---

### Dry Run (Preview Without Making Changes)

```bash
rsync -avzn ./local-dir/ user@hostname:/remote/path/
```

---

### Delete Files Not in Source

```bash
rsync -avz --delete ./local-dir/ user@hostname:/remote/path/
```

> ⚠️ Files on the remote that don't exist locally will be deleted.

---

### Exclude Files/Directories

```bash
rsync -avz --exclude='node_modules' --exclude='.git' ./local-dir/ user@hostname:/remote/path/
```

---

### Resume an Interrupted Transfer

```bash
rsync -avz --partial --progress ./large-file.tar.gz user@hostname:/remote/path/
```

---

### Sync Over a Specific SSH Port

```bash
rsync -avz -e "ssh -p 2222" ./local-dir/ user@hostname:/remote/path/
```

---

## Remote Commands

### Run a Single Command on Remote Server

```bash
ssh user@hostname "ls -la /var/www"
```

---

### Run Multiple Commands

```bash
ssh user@hostname "cd /var/www && git pull && npm install"
```

---

### Run a Local Script on Remote Server

```bash
ssh user@hostname 'bash -s' < ./local-script.sh
```

---

### Execute Command on Multiple Servers

```bash
for host in server1 server2 server3; do
  echo "=== $host ===" && ssh user@$host "uptime"
done
```

---

### Open Remote File in Local Editor

```bash
# Using VSCode Remote SSH extension
code --remote ssh-remote+hostname /path/to/file

# Using SSHFS to mount remote filesystem
sshfs user@hostname:/remote/path ~/mnt/remote
```

---

## SSH Agent

The SSH agent holds your decrypted keys in memory so you don't have to re-enter passphrases.

### Start SSH Agent

```bash
eval "$(ssh-agent -s)"
```

---

### Add Key to Agent

```bash
ssh-add ~/.ssh/id_ed25519
```

**Add all default keys:**
```bash
ssh-add
```

---

### List Keys in Agent

```bash
ssh-add -l
```

---

### Remove Key from Agent

```bash
ssh-add -d ~/.ssh/id_ed25519
```

**Remove all keys:**
```bash
ssh-add -D
```

---

### macOS Keychain Integration

Add to `~/.ssh/config` to persist keys across reboots:

```
Host *
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

---

## Troubleshooting

### Test SSH Connection

```bash
ssh -T git@github.com
```

---

### Check Server's Host Key

```bash
ssh-keyscan hostname
```

---

### Fix "Permissions too open" Error

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/config
```

---

### Remove Old Host Key (After Server Rebuild)

```bash
ssh-keygen -R hostname
```

---

### Check SSH Server Status (on remote)

```bash
sudo systemctl status sshd
```

---

### View SSH Login Attempts

```bash
# Linux
sudo journalctl -u sshd | grep "Failed password"

# macOS
sudo log show --predicate 'process == "sshd"' --last 1h
```
