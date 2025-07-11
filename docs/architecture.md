# SSH Honeypot Architecture

> Technical architecture design and system components for the isolated SSH honeypot environment.

## Architecture Overview

This honeypot system implements a three-tier architecture designed around complete network isolation, controlled attack simulation, and comprehensive data collection. The design prioritizes security containment while maintaining realistic attack surfaces.

## Network Architecture Design

### Dual Network Strategy

I designed the network with two completely separate paths to solve the fundamental challenge of needing both security isolation and internet connectivity.

**Network Tier 1: Host-Only Network (192.168.243.0/24)**
```
Purpose: Isolated attack simulation environment
Scope: VM-to-VM communication only
Security: Zero external connectivity
Gateway: 192.168.243.1 (VirtualBox interface)
```

**Network Tier 2: NAT Network (10.0.2.0/24)**
```
Purpose: Outbound internet access for updates
Scope: VM-to-Internet only
Security: Inbound connections blocked by NAT
Gateway: 10.0.2.1 (VirtualBox NAT)
```

### Traffic Flow Architecture

**Isolated Attack Path:**
```
Kali (192.168.243.3) ──Host-Only──► Ubuntu (192.168.243.4)
```

**Maintenance Path:**
```
Ubuntu (10.0.2.15) ──NAT──► Internet (Updates/Packages)
Kali (10.0.2.3) ──NAT──► Internet (Tool updates)
```

**Blocked Paths:**
```
Internet ──XXX──► Ubuntu (No inbound NAT)
Internet ──XXX──► Kali (No inbound NAT)
Host-Only Network ──XXX──► Host PC (VirtualBox isolation)
```

## System Architecture Components

### Honeypot Server Architecture

**Base Platform:**
- Ubuntu Server 22.04 LTS (minimal installation)
- 2GB RAM / 20GB storage / 1 CPU core
- Dual network interfaces (NAT + Host-Only)

**Application Stack:**
```
Port 22 ─► Python3 Process ─► Cowrie Framework
            │
            ├── SSH Protocol Handler
            ├── Authentication Manager  
            ├── Filesystem Emulator
            ├── Session Recorder
            └── Log Generator
```

**Security Architecture:**
```
Network Layer: UFW Firewall Rules
Process Layer: Non-root execution (cowrie user)
Port Layer: Linux capabilities (cap_net_bind_service)
File Layer: Restricted filesystem access
```

### Attack Platform Architecture

**Base Platform:**
- Kali Linux 2024.1
- 2GB RAM / 20GB storage / 1 CPU core  
- Single network interface (Host-Only)

**Tool Architecture:**
```
Attack Orchestration ─► Bash Scripts
                       │
                       ├── Nmap (Reconnaissance)
                       ├── Hydra (Brute Force)
                       ├── Custom Tools (Automation)
                       └── Data Collection
```

## Security Architecture Design

### Containment Strategy

**Layer 1: Hypervisor Isolation**
- VirtualBox provides hardware-level separation
- VM escape attacks contained by hypervisor
- Snapshot capability for instant recovery

**Layer 2: Network Segmentation**  
- Host-Only network prevents external exposure
- NAT network blocks inbound connections
- Firewall rules restrict traffic flow

**Layer 3: Process Isolation**
- Cowrie runs as unprivileged user
- Linux capabilities limit port binding scope
- Process sandboxing prevents privilege escalation

### Access Control Architecture

**Honeypot Access Model:**
```
SSH Port 22 ─► Cowrie Process ─► Fake Shell Environment
                    │
                    ├── No real system access
                    ├── Emulated commands only
                    ├── Logged interactions
                    └── Contained sessions
```

**Administrative Access Model:**
```
Host PC ─► VM Console ─► Root Account ─► System Management
            │
            ├── Direct hypervisor access
            ├── Full system control
            ├── Log collection
            └── Configuration management
```

## Data Architecture Design

### Log Collection Framework

**Primary Data Streams:**
```
SSH Connections ─► Cowrie Logger ─► /var/log/cowrie/cowrie.log
Authentication ─► Auth Module ─► Structured log entries  
Commands ─► Session Handler ─► Keystroke recording
Files ─► Download Monitor ─► Binary artifact storage
```

**Data Storage Architecture:**
```
Real-time Logs ─► Local Storage ─► Host PC Export
                      │
                      ├── Rotation policies
                      ├── Compression algorithms  
                      ├── Retention schedules
                      └── Backup procedures
```

### Intelligence Architecture

**Analysis Pipeline:**
```
Raw Logs ─► Pattern Detection ─► Threat Classification ─► Intelligence Reports
            │
            ├── Attack methodology analysis
            ├── Tool signature identification
            ├── Credential pattern analysis  
            └── Behavioral analytics
```

## Scalability Architecture

### Horizontal Scaling Design

**Multi-Honeypot Architecture:**
```
Load Balancer ─► Honeypot Cluster ─► Centralized Logging
                    │
                    ├── Multiple VM instances
                    ├── Distributed attack surface
                    ├── Geographic distribution
                    └── Fault tolerance
```

### Vertical Scaling Design

**Resource Expansion:**
- Memory scaling for high-volume attacks
- Storage scaling for extended logging
- CPU scaling for analysis workloads
- Network scaling for bandwidth requirements

## Recovery Architecture

### Backup Strategy Design

**VM-Level Backups:**
- Snapshot-based point-in-time recovery
- Template-based rapid deployment
- Configuration versioning
- State preservation

**Data-Level Backups:**
- Log file replication to host system
- Analysis result preservation
- Configuration file backup
- Evidence chain maintenance

### Disaster Recovery Design

**Recovery Procedures:**
1. Snapshot restoration for VM corruption
2. Template deployment for fresh instances
3. Log reconstruction from backups
4. Network reconfiguration automation

---

**Architecture Principles**: This design prioritizes isolation, monitoring, scalability, and maintainability while ensuring realistic attack simulation capabilities.
