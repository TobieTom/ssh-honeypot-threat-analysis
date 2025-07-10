# SSH Honeypot Setup Guide

This guide documents the actual process I went through to deploy a Cowrie SSH honeypot, including the real challenges I encountered and how I solved them.

## My Learning Journey

This was my first honeypot deployment. I learned that theory and practice are very different - what looks simple in tutorials becomes complex when you hit real-world issues.

## Environment Setup

### Virtual Machine Configuration

**What I built:**
- **Ubuntu Server 22.04** - Downloaded ISO and installed via VirtualBox
- **Kali Linux** - For simulating attacks against my honeypot
- **Network Setup**: Both VMs with dual adapters (NAT + Host-Only)

**Key lesson learned:** I initially tried Host-Only networking alone, but couldn't download packages. Had to use dual adapters - NAT for internet access, Host-Only for VM communication.

**My VM specs:**
- Ubuntu: 2GB RAM, 20GB storage, IP 192.168.243.4
- Kali: 2GB RAM, 20GB storage, IP 192.168.243.3

### Network Security - First Major Challenge

**The problem:** I needed internet access to download Cowrie, but also needed isolation for security.

**My solution:**
```bash
# Firewall rules I implemented
sudo ufw enable
sudo ufw default deny incoming  
sudo ufw allow from 192.168.243.0/24  # Only my lab network
```
**Why this worked:** VMs can reach internet through NAT, but external attackers can't reach them. Only my Kali VM can connect via Host-Only network.

**Honeypot Installation - The Real Process**
**User Account Setup**
**What I did:**
```bash
sudo adduser --disabled-password cowrie
sudo su - cowrie
```
**Why this matters:** I learned that running services as root is a security risk. Even in a lab, good habits matter.

**Cowrie Installation - Where Things Got Complicated**
**The download part was easy:**
```bash
git clone https://github.com/cowrie/cowrie.git
cd cowrie
```

**The virtual environment setup hit a snag:**
```bash
python3 -m venv cowrie-env
# This failed initially - missing packages
```

**Had to go back to main user and install system packages:**
```bash
exit  # Back to main user
sudo apt install python3-venv python3-dev build-essential libssl-dev libffi-dev
sudo su - cowrie  # Back to cowrie user
cd cowrie
python3 -m venv cowrie-env
source cowrie-env/bin/activate
pip install -r requirements.txt
```
**Lesson:** Virtual environments need system-level dependencies first.

**Configuration - Trial and Error**
**My first attempt at config (etc/cowrie.cfg):**
```ini
[honeypot]
hostname = backup-srv01

[ssh]
listen_endpoints = tcp:2222:interface=0.0.0.0
```
**Why port 2222 first:** Started here to avoid conflicts with any existing SSH service.

**The Filesystem Problem - My Biggest Challenge**
**What happened:** When I tested the honeypot, ls showed nothing. Empty directories made it obviously fake.
**My debugging process:**
1. Found TWO filesystem files: ./src/cowrie/data/fs.pickle and ./share/cowrie/fs.pickle
2. Discovered the default one had much more content
3. Had to update config to point to the right filesystem

**Final working config:**
```ini
[honeypot]
hostname = backup-srv01
filesystem_file = src/cowrie/data/fs.pickle

[ssh]
listen_endpoints = tcp:22:interface=0.0.0.0
```

**Port 22 Challenge - The setcap Discovery**
**The problem:** Port 22 needs root privileges, but I'm running as cowrie user.
**First attempt failed:**
```bash
sudo setcap cap_net_bind_service=+ep /usr/bin/python3
# Error: invalid argument
```

**The solution - I learned about symlinks:**
```bash
file $(which python3)
# Showed: /usr/bin/python3 is a symbolic link to python3.10

sudo setcap cap_net_bind_service=+ep /usr/bin/python3.10
# This worked!
```
**Key insight:** setcap needs the actual executable, not the symlink.

**Testing Process**
**My validation steps:**
```bash
# 1. Check if service is listening
sudo netstat -tlnp | grep :22

# 2. Test from Kali
ssh root@192.168.243.4
# Tried passwords: password123, admin, 123456

# 3. Monitor logs
tail -f var/log/cowrie/cowrie.log
```

**Real Problems I Encountered**
**SSH Service Not Found**
- Ubuntu Server didn't have SSH installed by default
- This was actually good - no conflicts with my honeypot

**Filesystem Loading Issues**
- Took multiple attempts to get the fake filesystem working
- Had to understand Cowrie's file structure

**Permission Problems**
-Learning about Linux capabilities was new to me
-Understanding symlinks vs actual executables

**What Makes This Setup Realistic**
**The honeypot appears as:** A small business backup server with weak security
- Hostname suggests backup purpose
- Responds on standard SSH port
- Has realistic Linux filesystem
- Shows typical security mistakes (weak passwords accepted)

**Current Status**
**Working honeypot that:**
- Accepts connections on port 22
- Logs all attacker activity
- Presents convincing Linux environment
- Safely contains all activity in VM








