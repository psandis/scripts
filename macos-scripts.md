# macOS Scripts & Commands

A curated collection of useful macOS-specific commands and scripts for everyday development and system tasks.

---

## Table of Contents

- [System Information](#system-information)
- [File & Directory](#file--directory)
- [Clipboard](#clipboard)
- [Networking](#networking)
- [Processes & Performance](#processes--performance)
- [Homebrew](#homebrew)
- [Finder & UI](#finder--ui)
- [Screenshots & Screen](#screenshots--screen)
- [Power & Battery](#power--battery)
- [Defaults & Preferences](#defaults--preferences)

---

## System Information

### macOS Version

```bash
sw_vers
```

**Just the version number:**
```bash
sw_vers -productVersion
```

---

### CPU Info

```bash
sysctl -n machdep.cpu.brand_string
```

---

### Total RAM

```bash
sysctl -n hw.memsize | awk '{print $1/1024/1024/1024 " GB"}'
```

---

### List Hardware Overview

```bash
system_profiler SPHardwareDataType
```

---

### Check macOS Architecture (Intel vs Apple Silicon)

```bash
uname -m
# arm64 = Apple Silicon, x86_64 = Intel
```

---

### Show All Environment Variables

```bash
env | sort
```

---

## File & Directory

### Show Hidden Files in Finder

```bash
defaults write com.apple.finder AppleShowAllFiles YES && killall Finder
```

**Hide them again:**
```bash
defaults write com.apple.finder AppleShowAllFiles NO && killall Finder
```

---

### Quick Look a File from Terminal

```bash
qlmanage -p file.txt
```

---

### Get File Info

```bash
mdls file.pdf
```

---

### Spotlight Search from Terminal

```bash
mdfind "query"
```

**Search in specific directory:**
```bash
mdfind -onlyin ~/Documents "query"
```

---

### Lock a File (Prevent Deletion)

```bash
chflags uchg file.txt
```

**Unlock:**
```bash
chflags nouchg file.txt
```

---

### Get/Set Extended Attributes

```bash
xattr -l file.txt
xattr -d com.apple.quarantine file.txt
```

> Useful for removing the quarantine flag from downloaded apps/files.

---

### Open File/Folder in Finder

```bash
open .
open ~/Downloads
open -a "Visual Studio Code" .
```

---

## Clipboard

### Copy to Clipboard

```bash
echo "Hello" | pbcopy
cat file.txt | pbcopy
```

---

### Paste from Clipboard

```bash
pbpaste
pbpaste > file.txt
```

---

### Copy File Contents to Clipboard

```bash
pbcopy < file.txt
```

---

## Networking

### Get Local IP Address

```bash
ipconfig getifaddr en0
```

**For Wi-Fi (en0) or Ethernet (en1):**
```bash
ifconfig en0 | grep "inet " | awk '{print $2}'
```

---

### Get Public IP Address

```bash
curl -s ifconfig.me
```

---

### Flush DNS Cache

```bash
sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder
```

---

### List Open Ports

```bash
sudo lsof -i -P | grep LISTEN
```

**Specific port:**
```bash
sudo lsof -i :3000
```

---

### Check Wi-Fi Network Name

```bash
networksetup -getairportnetwork en0
```

---

### List Available Wi-Fi Networks

```bash
/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -s
```

---

### Show Network Interface Stats

```bash
netstat -ib
```

---

### Ping with Timestamp

```bash
ping -i 1 google.com | while read line; do echo "$(date): $line"; done
```

---

## Processes & Performance

### List All Running Processes

```bash
ps aux
```

---

### Kill Process by Name

```bash
killall "App Name"
```

**Force kill:**
```bash
killall -9 "App Name"
```

---

### Find Process Using a Port

```bash
lsof -i :3000
```

**Kill it:**
```bash
lsof -ti :3000 | xargs kill -9
```

---

### Monitor CPU & Memory in Real-Time

```bash
top -o cpu
```

---

### Show Memory Pressure

```bash
memory_pressure
```

---

### List Startup Items

```bash
launchctl list
```

---

## Homebrew

### Update Homebrew and All Packages

```bash
brew update && brew upgrade
```

---

### Search for a Package

```bash
brew search node
```

---

### Install / Uninstall

```bash
brew install ripgrep
brew uninstall ripgrep
```

---

### List Installed Packages

```bash
brew list
```

---

### Check for Issues

```bash
brew doctor
```

---

### Clean Up Old Versions

```bash
brew cleanup
```

---

### Install a GUI App (Cask)

```bash
brew install --cask visual-studio-code
```

---

## Finder & UI

### Restart Finder

```bash
killall Finder
```

---

### Restart Dock

```bash
killall Dock
```

---

### Restart Menu Bar

```bash
killall SystemUIServer
```

---

### Add Spacer to Dock

```bash
defaults write com.apple.dock persistent-apps -array-add '{"tile-type"="spacer-tile";}' && killall Dock
```

---

### Show Path Bar in Finder

```bash
defaults write com.apple.finder ShowPathbar -bool true && killall Finder
```

---

### Show Status Bar in Finder

```bash
defaults write com.apple.finder ShowStatusBar -bool true && killall Finder
```

---

## Screenshots & Screen

### Take Screenshot (CLI)

```bash
screencapture screenshot.png
```

**After 5 second delay:**
```bash
screencapture -T 5 screenshot.png
```

**Specific window (interactive):**
```bash
screencapture -w screenshot.png
```

---

### Change Default Screenshot Location

```bash
defaults write com.apple.screencapture location ~/Pictures/Screenshots
killall SystemUIServer
```

---

### Get Display Resolution

```bash
system_profiler SPDisplaysDataType | grep Resolution
```

---

## Power & Battery

### Prevent Sleep (while command runs)

```bash
caffeinate -s
```

**For a specific duration (seconds):**
```bash
caffeinate -t 3600
```

**While running a command:**
```bash
caffeinate npm run build
```

---

### Check Battery Status

```bash
pmset -g batt
```

---

### Show Sleep/Wake History

```bash
pmset -g log | grep -E "Sleep|Wake"
```

---

### Shutdown / Restart / Sleep

```bash
sudo shutdown -h now     # Shutdown
sudo shutdown -r now     # Restart
pmset sleepnow           # Sleep
```

---

## Defaults & Preferences

### Disable Gatekeeper (allow apps from anywhere)

```bash
sudo spctl --master-disable
```

**Re-enable:**
```bash
sudo spctl --master-enable
```

---

### Disable Notification Center

```bash
launchctl unload -w /System/Library/LaunchAgents/com.apple.notificationcenterui.plist
```

---

### Speed Up Animations

```bash
defaults write NSGlobalDomain NSWindowResizeTime -float 0.001
```

---

### Enable Key Repeat (disable press-and-hold)

```bash
defaults write NSGlobalDomain ApplePressAndHoldEnabled -bool false
```

**Set repeat rate (lower = faster):**
```bash
defaults write NSGlobalDomain KeyRepeat -int 2
defaults write NSGlobalDomain InitialKeyRepeat -int 15
```

---

### Show Full Path in Finder Title

```bash
defaults write com.apple.finder _FXShowPosixPathInTitle -bool true && killall Finder
```

---

### Disable .DS_Store on Network Drives

```bash
defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool true
```
