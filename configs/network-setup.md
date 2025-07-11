# Network Setup Guide

> Complete network configuration guide for the isolated SSH honeypot environment, including the real challenges I faced and solutions I implemented.

## The Network Challenge I Had to Solve

When I started this project, I thought network setup would be the easy part. Just create some VMs and connect them, right? Wrong. I quickly discovered that building a secure, isolated lab environment while maintaining internet connectivity is more complex than it looks.

**My requirements:**
- VMs need to communicate with each other (for attack simulation)
- VMs need internet access (for updates and package installation)
- Honeypot must be completely isolated from external attackers
- No risk to my host computer or home network

**The problem:** These requirements seemed contradictory at first.

## My Network Architecture Solution

After trial and error, I implemented a dual-network approach that provides both security and functionality:

```
Internet
    │
    │ (NAT - Outbound Only)
    │
┌───▼──────────────────────────────────────┐
│           VirtualBox Host                │
│                                          │
│  ┌─────────────┐    ┌─────────────┐      │
│  │ Kali Linux  │    │Ubuntu Server│      │
│  │(Attacker VM)│    │(Honeypot VM)│      │
│  │             │    │             │      │
│  │10.0.2.3     │    │10.0.2.15    │      │ NAT Network
│  │(Internet)   │    │(Internet)   │      │ (Updates/Packages)
│  │             │    │             │      │
│  │192.168.243.3│◄──►│192.168.243.4│      │ Host-Only Network
│  │(Lab Only)   │    │(Lab Only)   │      │ (Attack Traffic)
│  └─────────────┘    └─────────────┘      │
└───────────────────────────────────────────┘
```

## Network Implementation Details

### Host-Only Network Configuration

**Purpose:** Isolated attack simulation environment
**Network Range:** 192.168.243.0/24 (VirtualBox assigned this automatically)
**Gateway:** 192.168.243.1 (VirtualBox host interface)

**Why Host-Only:**
- Complete isolation from external networks
- VMs can communicate with each other
- No internet routing possible
- Host computer maintains control

**VirtualBox Configuration:**
1. File → Host Network Manager (or Tools → Network)
2. Create new Host-Only network (if not exists)
3. Configure IP range: 192.168.243.0/24
4. DHCP can be enabled or manual IPs assigned

### NAT Network Configuration

**Purpose:** Outbound internet access for maintenance
**Network Range:** 10.0.2.0/24 (VirtualBox default)
**Gateway:** 10.0.2.1 (VirtualBox NAT gateway)

**Why NAT:**
- Allows outbound connections (updates, packages)
- Blocks all inbound connections automatically
- No port forwarding configured (security)
- Transparent to the VMs

**Security Features:**
- No external visibility of internal IPs
- Automatic firewall protection
- No direct internet exposure

## VM Network Adapter Configuration

### Ubuntu Server (Honeypot) - Dual Adapters

**Adapter 1: NAT (Internet Access)**
```
Type: NAT
MAC: Auto-generated
IP: 10.0.2.15 (DHCP assigned)
Purpose: Package downloads, updates
Interface: enp0s3 (first network card)
```

**Adapter 2: Host-Only (Lab Traffic)**
```
Type: Host-only Adapter
Name: vboxnet0 (or similar)
IP: 192.168.243.4 (static or DHCP)
Purpose: Honeypot traffic, SSH attacks
Interface: enp0s8 (second network card)
```

### Kali Linux (Attacker) - Single Adapter

**Adapter 1: Host-Only (Attack Platform)**
```
Type: Host-only Adapter
Name: vboxnet0 (same as Ubuntu)
IP: 192.168.243.3 (static or DHCP)
Purpose: Launch attacks against honeypot
Interface: eth0
```

**Note:** Kali also gets internet through NAT, but I configured only Host-Only for attack traffic focus.

## Step-by-Step Network Setup

### Step 1: Configure VirtualBox Host Networks

**Create Host-Only Network:**
1. Open VirtualBox Manager
2. File → Host Network Manager
3. Click "Create" if no host-only networks exist
4. Note the network name (e.g., vboxnet0)
5. Configure IP range: 192.168.243.0/24

**Verify Network Creation:**
```bash
# On host computer (Windows/Mac/Linux)
# Check for vboxnet interfaces
ip addr show  # Linux/Mac
ipconfig /all  # Windows
```

### Step 2: Configure Ubuntu Server VM

**Network Adapter Settings:**
1. Select Ubuntu VM → Settings → Network
2. Adapter 1: Enable, set to NAT
3. Adapter 2: Enable, set to Host-only Adapter
4. Select the host-only network created in Step 1
5. Click OK

**Verify Network Configuration:**
```bash
# Inside Ubuntu VM
ip addr show
# Should see two interfaces:
# enp0s3: 10.0.2.x (NAT network)
# enp0s8: 192.168.243.x (Host-Only network)

# Test internet connectivity
ping google.com  # Should work through NAT

# Check routing table
ip route show
# Should show default route through 10.0.2.1
```

### Step 3: Configure Kali Linux VM

**Network Adapter Settings:**
1. Select Kali VM → Settings → Network
2. Adapter 1: Enable, set to Host-only Adapter
3. Select the same host-only network as Ubuntu
4. Adapter 2: (Optional) NAT for internet access
5. Click OK

**Verify Attack Connectivity:**
```bash
# Inside Kali VM
ip addr show
# Should see: eth0: 192.168.243.x

# Test connectivity to honeypot
ping 192.168.243.4  # Should reach Ubuntu honeypot

# Test SSH connectivity
ssh root@192.168.243.4  # Should connect to Cowrie
```

## Network Troubleshooting Guide

### Problem: VMs Can't Communicate

**Symptoms:** `ping` between VMs fails, SSH connection refused

**Diagnosis:**
```bash
# Check IP addresses
ip addr show

# Check if both VMs are on same Host-Only network
# IPs should be in same range (192.168.243.x)

# Check routing
ip route show

# Check firewall
sudo ufw status
```

**Solution:**
- Verify both VMs use same Host-Only network adapter
- Check firewall rules (see firewall-rules.txt)
- Restart network services: `sudo systemctl restart networking`

### Problem: No Internet Access

**Symptoms:** Can't download packages, `ping google.com` fails

**Diagnosis:**
```bash
# Check NAT adapter status
ip addr show enp0s3  # Should have 10.0.2.x IP

# Check default route
ip route show | grep default  # Should point to 10.0.2.1

# Test DNS resolution
nslookup google.com
```

**Solution:**
- Verify NAT adapter is enabled in VM settings
- Check if DHCP client is running: `sudo dhclient enp0s3`
- Restart networking: `sudo systemctl restart networking`

### Problem: Wrong IP Range

**Symptoms:** VMs get unexpected IP addresses

**This happened to me** - VirtualBox assigned 192.168.243.x instead of the 192.168.56.x I expected from tutorials.

**Solution:**
- Use whatever range VirtualBox assigns
- Update firewall rules to match actual range
- Document the actual range in your setup

**Commands to find actual range:**
```bash
# On Ubuntu honeypot
ip addr show enp0s8

# On Kali attacker  
ip addr show eth0

# Update firewall rules with correct range
sudo ufw allow from 192.168.243.0/24  # Use actual range
```

## Network Security Verification

### Test 1: Isolation Verification

**From host computer:**
```bash
# Try to reach honeypot (should fail)
ping 192.168.243.4  # Should timeout or "no route"
ssh root@192.168.243.4  # Should fail to connect
```

**Expected result:** Host computer cannot reach VMs on Host-Only network

### Test 2: Lab Connectivity

**From Kali VM:**
```bash
# Should successfully reach honeypot
ping 192.168.243.4  # Should get replies
ssh root@192.168.243.4  # Should connect to Cowrie
nmap 192.168.243.4  # Should show port 22 open
```

**Expected result:** Attack VM can reach honeypot VM

### Test 3: Internet Access

**From both VMs:**
```bash
# Should work for updates/packages
ping 8.8.8.8  # Should get replies
curl -I google.com  # Should return HTTP headers
sudo apt update  # Should download package lists
```

**Expected result:** Both VMs can reach internet for maintenance

## Network Performance Considerations

### Resource Allocation

**For 2GB RAM per VM:**
- Network adapters use minimal RAM
- Host-Only networking is faster than NAT
- Multiple adapters add slight CPU overhead

**Optimization tips:**
- Disable unnecessary network services
- Use lightweight network monitoring
- Keep firewall rules simple

### Bandwidth Considerations

**Host-Only Network:**
- Very fast (memory-speed communication)
- No internet bandwidth usage
- Ideal for high-volume attack simulation

**NAT Network:**
- Limited by host internet connection
- Shared bandwidth between VMs
- Fine for package updates and maintenance

## Advanced Network Configuration

### Static IP Assignment

**If you want consistent IPs:**

**Ubuntu (/etc/netplan/01-netcfg.yaml):**
```yaml
network:
  version: 2
  ethernets:
    enp0s3:  # NAT adapter
      dhcp4: true
    enp0s8:  # Host-Only adapter
      addresses: [192.168.243.4/24]
      dhcp4: false
```

**Apply changes:**
```bash
sudo netplan apply
```

### Network Monitoring

**Monitor network traffic:**
```bash
# Watch network interfaces
watch -n 1 'ip -s link'

# Monitor connections
sudo netstat -tulpn

# Watch attack traffic
sudo tcpdump -i enp0s8 port 22
```

## Lessons Learned

### What Surprised Me

**VirtualBox network complexity:** I thought "just connect the VMs" would be simple. Reality: had to understand NAT, Host-Only, bridged, and internal network types.

**IP range assignment:** Tutorials showed 192.168.56.x, but VirtualBox gave me 192.168.243.x. Had to adapt all my firewall rules.

**Dual adapter necessity:** Tried Host-Only first, couldn't download packages. Learned that security isolation requires careful network design.

### What I'd Do Differently

**Plan IP ranges first:** Should have documented the network layout before configuring VMs.

**Test connectivity early:** Should have verified VM-to-VM communication before installing honeypot software.

**Snapshot after networking:** Should have created VM snapshots after getting networking right.

### Key Takeaways

**Network isolation is crucial** for honeypot security - but requires thoughtful design to maintain functionality.

**VirtualBox networking is powerful** but requires understanding different network types and their security implications.

**Documentation is essential** because network configuration is easy to forget and hard to recreate.

---

**This network setup took about 2 hours to get right**, including research, trial and error, and testing. The final configuration provides both security and functionality for safe honeypot operations.
