# Homelab Infrastructure

**Production-grade home server environment demonstrating enterprise DevOps practices and modern infrastructure patterns**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)](https://www.docker.com/)
[![Linux](https://img.shields.io/badge/OS-Debian%2013-A81D33?logo=debian)](https://www.debian.org/)
[![Network](https://img.shields.io/badge/Router-OpenWRT-00B5E2?logo=openwrt)](https://openwrt.org/)
[![Monitoring](https://img.shields.io/badge/Monitoring-Prometheus%20%2B%20Grafana-E6522C?logo=prometheus)](https://prometheus.io/)

---

## Table of Contents

- [Overview](#overview)
- [Network Architecture](#network-architecture)
- [Infrastructure Components](#infrastructure-components)
- [Services & Applications](#services--applications)
- [Monitoring & Observability](#monitoring--observability)
- [Remote Access](#remote-access)
- [Hardware Specifications](#hardware-specifications)
- [Skills Demonstrated](#skills-demonstrated)
- [Future Enhancements](#future-enhancements)
- [Documentation](#documentation)

---

## Overview

This repository documents a production homelab environment built on Debian 13, showcasing enterprise-grade infrastructure practices including containerized service orchestration, comprehensive monitoring, automated backups, and secure remote access. The infrastructure serves as both a learning platform and a functional media/cloud services deployment.

**Core Principles:**
- **Infrastructure as Code:** All services defined declaratively via Docker Compose
- **Observability First:** Comprehensive metrics collection and visualization
- **Security by Design:** Defense-in-depth with VPN mesh networking and zero-trust access
- **Automated Operations:** Self-healing services with automated updates and backups
- **Scalable Architecture:** Modular design supporting future expansion

**Service Categories:**
- **24 containerized services** across 4 functional stacks
- **Media automation** with *arr ecosystem + Jellyfin
- **Cloud services** including photo management and file sync
- **Smart home** integration with Home Assistant
- **Full-stack monitoring** with Prometheus + Grafana

**External Access:**
- Jellyfin Media Server: [jellyfin.unfunky.xyz](https://jellyfin.unfunky.xyz) (Cloudflare Tunnel)
- Private services accessible via Tailscale mesh VPN

---

## Network Architecture

### Topology Overview

```
Internet (Xfinity ISP / Dynamic IP with DDNS)
    ↓
Cloudflare (DNS Management + Zero Trust Gateway + DDoS Protection)
    ↓
ASUS RT-AC3100 Router (OpenWRT 23.05, Gigabit Routing, Dual-band WiFi)
    ↓
Primary Network (192.168.1.0/24)
    ├─ SFF Debian Server (192.168.1.29) - Main infrastructure host
    ├─ Raspberry Pi 3B+ - Moode Audio Server
    └─ Trusted devices, workstations, mobile devices
    ↓
Docker Network Architecture (4 isolated networks)
    ├─ proxy_network    - Reverse proxy & management services
    ├─ media_network    - Media automation stack (*arr apps)
    ├─ cloud_network    - Cloud services & databases
    └─ monitoring_network - Metrics collection & visualization
    ↓
Tailscale Mesh VPN (WireGuard-based, 5+ devices)
    └─ Encrypted remote access to all services
```

### Network Design Rationale

| Component | Technology Choice | Justification |
|-----------|------------------|---------------|
| **Router Firmware** | OpenWRT 23.05 | Superior control over routing, firewall rules, and advanced network features compared to stock firmware |
| **VPN Architecture** | Tailscale WireGuard mesh | Zero-configuration P2P encrypted access, no port forwarding required, sub-50ms latency |
| **Service Isolation** | Docker network segmentation | Logical separation of service stacks, improved security posture, simplified firewall rules |
| **Container Runtime** | Docker with Compose V2 | Industry-standard orchestration, simplified dependency management, rapid deployment/rollback |
| **Public Access** | Cloudflare Tunnel | Eliminates port forwarding, DDoS protection, SSL/TLS automation, zero-trust access control |

### Planned Network Enhancements

**VLAN Segmentation (In Progress):**
- **VLAN 1 (Production):** Server infrastructure, trusted devices
- **VLAN 2 (IoT/Untrusted):** Smart home devices, network-isolated
- **Goal:** Prevent lateral movement from compromised IoT devices to production infrastructure

---

## Infrastructure Components

### Network Gateway

**ASUS RT-AC3100 (OpenWRT 23.05)**
- Custom OpenWRT firmware for advanced network control
- Dual-band WiFi: 2.4GHz (1000 Mbps) + 5GHz (2167 Mbps)
- Hardware: Broadcom BCM4709C0 (1.4GHz dual-core ARM), 512MB RAM
- 4x Gigabit LAN ports + 1x Gigabit WAN
- DDNS integration for dynamic IP management
- Future: VLAN tagging and inter-VLAN firewall rules

### Physical Hosts

#### Primary Server: Custom SFF x86 System

| Component | Specification | Purpose |
|-----------|---------------|---------|
| **Motherboard** | MSI B450 | AMD AM4 platform, PCIe for GPU |
| **Processor** | AMD Ryzen 5 2600 (6C/12T @ 3.4GHz) | Multi-threaded workload handling |
| **Memory** | 16GB DDR4 | Container orchestration, in-memory caching |
| **Boot Drive** | 120GB SSD | OS + Docker system volumes |
| **Data Drive** | 6TB HDD | Media library storage |
| **GPU** | AMD Radeon R5 340 | Hardware-accelerated transcoding (VA-API) |
| **OS** | Debian 13 (Trixie) - Headless | Stable, long-term support (LTS ~2031) |
| **Network** | Gigabit Ethernet | 1000 Mbps wired connection |
| **Power** | 80W idle / 125W load | ~$105/year @ $0.15/kWh |
| **Uptime** | 99.5%+ | Prometheus-monitored availability |

**Storage Architecture:**
```
/dev/sda (120GB SSD) - LVM Layout
├─ /boot/efi     (976MB)   - UEFI boot partition
├─ /boot         (977MB)   - Kernel and initramfs
└─ LVM VG        (109.9GB)
   ├─ root       (81.3GB)  - Root filesystem
   └─ swap       (4.4GB)   - Swap space

/dev/sdb1 (5.5TB HDD) - Media Storage
└─ /mnt/storage
   ├─ media/              - Jellyfin library
   │  ├─ movies/
   │  ├─ tv/
   │  ├─ music/
   │  ├─ books/
   │  └─ photos/          - Immich photo library
   ├─ downloads/          - SABnzbd staging
   │  ├─ complete/
   │  └─ incomplete/
   ├─ appdata/            - Persistent container data
   │  ├─ nextcloud/
   │  ├─ immich/
   │  ├─ homeassistant/
   │  └─ postgresql/
   └─ backups/            - Automated backup storage
```

**Docker Configuration Storage:**
```
/opt/docker-configs/   - Service configurations (SSD for fast access)
~/docker/              - Docker Compose definitions
  ├─ core/             - Management services
  ├─ media/            - Media automation stack
  ├─ cloud/            - Cloud services
  └─ monitoring/       - Metrics collection
```

**Network Configuration:**
```
Interface    IP Address        Purpose
---------    ----------        -------
eth0         192.168.1.29/24   Primary network (static)
tailscale0   100.x.x.x/32      Mesh VPN (dynamic)
docker0      172.17.0.1/16     Default Docker bridge
```

**Build History:**
- Initial deployment: Debian 12 (Bookworm) → Migrated to Debian 13 (Trixie)
- Storage expansion: Added 6TB HDD for media library
- GPU addition: AMD R5 340 for Jellyfin hardware transcoding
- Network upgrade: Tailscale mesh VPN integration

#### Secondary Host: Raspberry Pi 3B+

| Component | Specification | Purpose |
|-----------|---------------|---------|
| **Model** | Raspberry Pi 3B+ | ARM-based single-board computer |
| **Purpose** | Moode Audio Server | Networked audio streaming appliance |
| **Network** | Wired Ethernet | Low-latency audio streaming |
| **Function** | High-fidelity music streaming to legacy audio equipment | |
| **Integration** | Connected to production network | Accessible via Tailscale mesh |

---

## Services & Applications

**Deployment Model:** All services run as Docker containers, managed via Docker Compose with Portainer for orchestration and monitoring.

### Media Automation Stack (8 containers)

| Service | Container | Port | Function | External Access |
|---------|-----------|------|----------|-----------------|
| **Jellyfin** | jellyfin | 8096 | Media streaming server with GPU transcoding | **Public** (Cloudflare Tunnel) |
| **Jellyseerr** | jellyseerr | 5055 | Media request and discovery platform | Tailscale VPN |
| **Sonarr** | sonarr | 8989 | TV series automation and library management | Internal |
| **Radarr** | radarr | 7878 | Movie automation and library management | Internal |
| **Prowlarr** | prowlarr | 9696 | Centralized indexer manager for *arr ecosystem | Internal |
| **Lidarr** | lidarr | 8686 | Music automation and library management | Internal |
| **Readarr** | readarr | 8787 | Book/audiobook automation and library management | Internal |
| **SABnzbd** | sabnzbd | 8080 | Usenet download client with category routing | Internal |

**Integration Flow:**
```
Jellyseerr (User Request) 
    ↓
Sonarr/Radarr (Search for content via Prowlarr)
    ↓
SABnzbd (Download from Usenet)
    ↓
Sonarr/Radarr (Process, rename, move to media library)
    ↓
Jellyfin (Media appears in library, ready for streaming)
```

**Jellyfin Hardware Transcoding:**
- GPU: AMD Radeon R5 340
- API: VA-API (Video Acceleration API)
- Supported codecs: H.264, HEVC, VC1, VP8, VP9
- Performance: 3-4 concurrent 1080p transcodes
- Power efficiency: ~30W under transcode load vs ~80W CPU-only

### Cloud Services Stack (7 containers)

| Service | Container | Port | Function | Data Storage |
|---------|-----------|------|----------|--------------|
| **Nextcloud** | nextcloud | 8888 | File sync, sharing, and collaboration platform | /mnt/storage/appdata/nextcloud |
| **Nextcloud DB** | nextcloud-db | - | PostgreSQL database for Nextcloud | Persistent volume |
| **Nextcloud Redis** | nextcloud-redis | - | In-memory cache for performance | Ephemeral |
| **Immich** | immich-server | 2283 | AI-powered photo management and organization | /mnt/storage/media/photos |
| **Immich Microservices** | immich-microservices | - | Background job processing (thumbnails, ML) | Shared volume |
| **Immich ML** | immich-machine-learning | - | Facial recognition and object detection | Model cache |
| **Immich DB** | immich-postgres | - | PostgreSQL + pgvector for embeddings | Persistent volume |

**Nextcloud Features:**
- File synchronization across devices
- Calendar and contact management
- Document collaboration (Office integration)
- End-to-end encryption support
- Mobile apps for iOS/Android

**Immich Features:**
- AI-powered facial recognition
- Object and scene detection
- Geographic photo clustering
- Automatic backup from mobile devices
- Privacy-focused (self-hosted alternative to Google Photos)

### Smart Home Integration (1 container)

| Service | Container | Port | Function | Network Mode |
|---------|-----------|------|----------|--------------|
| **Home Assistant** | homeassistant | 8123 | Home automation platform and device orchestration | **Host mode** (for device discovery) |

**Features:**
- IoT device integration and control
- Automation rule engine
- Voice assistant integration (planned)
- Energy monitoring and analytics
- Local control (no cloud dependency)

**Note:** Runs in host network mode to enable mDNS/discovery protocols for smart home devices.

### Infrastructure & Management (5 containers)

| Service | Container | Port | Function | Access Method |
|---------|-----------|------|----------|---------------|
| **Nginx Proxy Manager** | nginx-proxy-manager | 81 | Reverse proxy with automatic SSL/TLS certificate management | Internal + Tailscale |
| **Portainer** | portainer | 9000 | Docker container orchestration and management UI | Internal + Tailscale |
| **Heimdall** | heimdall | 8081 | Application dashboard and service directory | **Planned Public** (Cloudflare Zero Trust) |
| **Watchtower** | watchtower | - | Automated container image updates with Telegram notifications | Background service |
| **Cloudflared** | cloudflared | - | Cloudflare Tunnel for secure public access (Jellyfin) | Background service |

**Watchtower Configuration:**
- Update schedule: Daily at 4:00 AM
- Telegram notifications on container updates
- Cleanup old images after successful updates
- Monitors all containers for new image versions

**Nginx Proxy Manager:**
- Automatic Let's Encrypt SSL/TLS certificates
- Wildcard certificate support (*.unfunky.xyz)
- Cloudflare DNS challenge integration
- Access control lists and authentication

### Monitoring Stack (4 containers)

| Service | Container | Port | Function | Scrape Interval |
|---------|-----------|------|----------|-----------------|
| **Prometheus** | prometheus | 9090 | Time-series metrics database and alerting engine | 15 seconds |
| **Grafana** | grafana | 3001 | Metrics visualization and dashboard platform | N/A |
| **Node Exporter** | node-exporter | 9100 | System-level metrics (CPU, RAM, disk, network) | Scraped by Prometheus |
| **cAdvisor** | cadvisor | 9200 | Container-level resource monitoring | Scraped by Prometheus |

**Metrics Collected:**
- System: CPU usage (per-core), memory utilization, disk I/O, network throughput
- Containers: Per-container CPU/memory, network I/O, filesystem usage, restart counts
- Docker: Total container count, image storage, volume usage
- Application: Service-specific metrics (where exporters available)

### Game Servers (1 container)

| Service | Container | Port | Function | Players |
|---------|-----------|------|----------|---------|
| **Minecraft Server** | minecraft-forge | 25565 | Modded Forge multiplayer server | 2-8 concurrent |

**Configuration:**
- Server type: Forge (modded)
- Port forwarding: 25565/TCP (router configured)
- Backups: Daily world saves to /mnt/storage/backups
- Mods: Custom modpack (managed via Portainer)

---

## Monitoring & Observability

### Monitoring Architecture

**Philosophy:** Comprehensive observability through metrics collection, visualization, and alerting.

**Data Pipeline:**
```
System Resources → Node Exporter (port 9100) ──┐
Docker Containers → cAdvisor (port 9200) ──────┤
Custom Metrics → (Future: Custom exporters) ───┼→ Prometheus (port 9090) → Grafana (port 3001)
                                                │
Application Logs → (Future: Loki + Promtail) ──┘
```

### Grafana Dashboards

**1. Node Exporter Full (Dashboard ID 1860)**
- **Purpose:** Comprehensive system-level monitoring
- **Metrics:**
  - CPU utilization (aggregate + per-core breakdown)
  - Memory usage, available capacity, swap utilization
  - Disk I/O rates (read/write IOPS and throughput)
  - Network traffic (bandwidth, packet rates, errors)
  - Filesystem usage and inode consumption
  - System load averages (1m, 5m, 15m)
  - Temperature sensors (CPU, motherboard)
- **Update Frequency:** 15-second granularity
- **Historical Data:** 30-day retention

**2. Docker Container Monitoring (Dashboard ID 893)**
- **Purpose:** Per-container resource tracking
- **Metrics:**
  - CPU usage per container (% of total system)
  - Memory allocation and consumption
  - Network I/O statistics (sent/received bytes)
  - Filesystem read/write operations
  - Container health status and restart counts
  - Image sizes and storage consumption
- **Coverage:** All 24 active containers
- **Alerting:** Visual indicators for resource-constrained containers

**3. System Overview (Custom)**
- **Purpose:** High-level infrastructure health dashboard
- **Panels:**
  - Service uptime status (all 24 containers)
  - Disk space utilization (boot drive + media drive)
  - Network bandwidth usage (7-day trend)
  - Container restart frequency
  - Quick links to service UIs
- **Use Case:** At-a-glance health check, daily monitoring

### Planned Monitoring Enhancements

**Security Monitoring (In Progress):**
- Fail2Ban Prometheus exporter (IP ban metrics)
- GeoIP attack mapping (geographic visualization of blocked IPs)
- SSH authentication attempt tracking
- Real-time Telegram alerts on security events

**Application Performance Monitoring:**
- Jellyfin playback metrics (concurrent streams, transcoding load)
- SABnzbd download queue statistics
- Database query performance (Nextcloud, Immich)
- Response time tracking for web services

**Alert System:**
- Grafana alerting rules for threshold violations
- Telegram bot integration for real-time notifications
- Alert categories: Critical (immediate), Warning (review), Info (logged)
- Planned thresholds:
  - CPU > 80% sustained for 5 minutes
  - Memory > 90% available
  - Disk > 85% full
  - Container restart loops (3+ restarts in 10 minutes)

---

## Remote Access

### Tailscale VPN Mesh Network

**Architecture:** WireGuard-based peer-to-peer encrypted mesh network connecting 5+ devices

**Connected Nodes:**
- Debian server (designated exit node for internet routing)
- Personal laptop (primary administration interface)
- Mobile device (iOS - remote monitoring)
- Family member devices (limited access scope)
- Additional trusted devices

**Network Topology:**
```
Tailscale Coordination Server (cloud-hosted)
    ↓
WireGuard Encrypted Mesh (automatic peering)
    ├─ Debian Server (100.x.x.1) - Exit node
    ├─ Laptop (100.x.x.2)
    ├─ Mobile (100.x.x.3)
    └─ Other Devices (100.x.x.4+)
```

**Advantages:**
- **Zero Configuration:** Automatic NAT traversal, no manual port forwarding
- **Low Latency:** Direct peer-to-peer connections when network topology permits
- **High Security:** WireGuard cryptography, automatic key rotation
- **Reliability:** Measured latency <50ms for local network access
- **Simplicity:** Single command deployment, cross-platform compatibility

**Use Cases:**
- Secure SSH access to server infrastructure from anywhere
- Access to internal web services (Portainer, Grafana, *arr apps)
- Remote media streaming via Jellyfin (Cloudflare Tunnel alternative)
- Network-level access to entire homelab from remote locations
- Exit node functionality for secure internet browsing when traveling

### Public Web Access

**Jellyfin Media Server:**
- **URL:** https://jellyfin.unfunky.xyz
- **Access Method:** Cloudflare Tunnel (cloudflared container)
- **Security Layers:**
  1. Cloudflare DDoS protection and WAF
  2. Zero Trust application authentication (planned)
  3. Jellyfin user authentication
  4. SSL/TLS encryption (automatic certificate management)
- **Performance:** CDN-accelerated delivery, sub-200ms latency globally

**Heimdall Dashboard (Planned):**
- **URL:** https://home.unfunky.xyz (pending deployment)
- **Authentication:** Cloudflare Zero Trust with email verification
- **Function:** Centralized portal for accessing internal services
- **Security:** Multi-layer authentication (Cloudflare → Heimdall)

### Access Control Matrix

| Service | Local Network | Tailscale VPN | Public Internet | Authentication |
|---------|---------------|---------------|-----------------|----------------|
| Jellyfin | ✅ Direct | ✅ Direct | ✅ Cloudflare Tunnel | User login |
| Portainer | ✅ Direct | ✅ Direct | ❌ Blocked | User login |
| Grafana | ✅ Direct | ✅ Direct | ❌ Blocked | Admin login |
| Nextcloud | ✅ Direct | ✅ Direct | ❌ Blocked | User login |
| Immich | ✅ Direct | ✅ Direct | ❌ Blocked | User login |
| Home Assistant | ✅ Direct | ✅ Direct | ❌ Blocked | User login |
| Sonarr/Radarr | ✅ Direct | ✅ Direct | ❌ Blocked | API key |
| SSH (port 22) | ✅ Key-based | ✅ Key-based | ❌ Blocked | SSH keys only |

---

## Hardware Specifications

### Server Hardware Details

**Form Factor:** Small Form Factor (SFF) for space-efficient deployment

**Chassis & Cooling:**
- Compact tower case with optimized airflow
- Aftermarket CPU cooler (Cooler Master Hyper 212)
- 3x 120mm case fans (intake + exhaust configuration)
- Thermal performance:
  - Idle: ~45°C CPU, ~40°C GPU
  - Load: ~65°C CPU, ~55°C GPU
  - Ambient temp compensation

**Power Consumption Breakdown:**
- Idle state: ~80W (system + HDD spin)
- Media streaming (no transcode): ~90W
- Single transcode (GPU): ~110W
- Multiple transcodes + downloads: ~125W peak
- Annual power cost: ~$105 @ $0.15/kWh (assuming 80% idle, 20% active)

**Reliability Metrics:**
- Uptime: 99.5%+ (measured via Prometheus)
- Mean time between failures (MTBF): No hardware failures in 12+ months
- Unplanned downtime: <4 hours/month (primarily OS updates)
- Planned maintenance windows: Monthly (system updates, testing)

**Storage Performance:**
- Boot SSD sequential read/write: ~500 MB/s / ~450 MB/s
- Media HDD sequential read/write: ~180 MB/s / ~160 MB/s
- Docker volume I/O: Direct SSD access (optimal for databases)
- Media streaming: HDD sufficient for 10+ concurrent streams

### Network Equipment Details

**ASUS RT-AC3100 (OpenWRT 23.05)**
- **Chipset:** Broadcom BCM4709C0 (1.4GHz dual-core ARM Cortex-A9)
- **Memory:** 512MB DDR3 RAM, 128MB NAND flash storage
- **WiFi Capabilities:**
  - 2.4GHz: 802.11n (up to 1000 Mbps)
  - 5GHz: 802.11ac (up to 2167 Mbps)
  - Simultaneous dual-band operation
- **Ethernet:** 4x Gigabit LAN ports, 1x Gigabit WAN port
- **Features:**
  - QoS traffic prioritization
  - DDNS integration (No-IP, DuckDNS)
  - UPnP for automatic port mapping
  - Advanced firewall with custom iptables rules
  - Future: VLAN tagging support

**Network Performance:**
- WAN-LAN throughput: ~940 Mbps (wire speed)
- WiFi throughput: ~600 Mbps (5GHz, ideal conditions)
- NAT table capacity: 65,536 simultaneous connections
- Uptime: 99.9%+ (router reboots only for firmware updates)

---

## Skills Demonstrated

This homelab infrastructure showcases proficiency across multiple technical domains:

### Systems Administration
- **Linux Server Management:** Debian 13 installation, configuration, and maintenance
- **Storage Management:** LVM configuration, filesystem optimization, automated mounting
- **User & Permission Management:** Sudo configuration, group membership, file permissions
- **System Monitoring:** Resource tracking, performance optimization, uptime management
- **Security Hardening:** SSH key-based authentication, firewall configuration (UFW), user isolation

### Containerization & Orchestration
- **Docker Fundamentals:** Image management, container lifecycle, volume/network configuration
- **Docker Compose:** Multi-container application definitions, service dependencies, network isolation
- **Service Deployment:** 24-container infrastructure with declarative configuration
- **Container Networking:** Custom bridge networks, service discovery, inter-container communication
- **Resource Management:** CPU/memory limits, restart policies, health checks

### Networking
- **Network Architecture:** LAN design, static IP assignment, DHCP reservation
- **Router Configuration:** OpenWRT firmware, advanced routing, port forwarding
- **VPN Technologies:** Tailscale mesh network deployment, WireGuard protocol understanding
- **Reverse Proxy:** Nginx Proxy Manager configuration, SSL/TLS certificate automation
- **DNS Management:** Cloudflare integration, DDNS for dynamic IP handling
- **Future:** VLAN segmentation, inter-VLAN firewall rules, network isolation

### Monitoring & Observability
- **Metrics Collection:** Prometheus time-series database configuration and tuning
- **Visualization:** Grafana dashboard design, panel creation, query optimization
- **System Metrics:** Node Exporter deployment, custom metric collection
- **Container Monitoring:** cAdvisor integration, per-container resource tracking
- **Data Retention:** Prometheus storage configuration, historical data management
- **Planned:** Alert rule creation, notification channels (Telegram, email)

### Security
- **Access Control:** Multi-layer authentication, principle of least privilege
- **Network Security:** Firewall rule design, port management, service isolation
- **Encryption:** SSL/TLS certificate management, VPN encryption (WireGuard)
- **SSH Hardening:** Key-based authentication, root login disabled, fail2ban (planned)
- **Zero Trust Architecture:** Cloudflare Zero Trust integration (in progress)
- **Planned:** Intrusion detection (Fail2Ban), geographic IP blocking, security metrics

### Cloud & DevOps Practices
- **Infrastructure as Code:** Declarative service definitions, version-controlled configurations
- **Automated Deployment:** Docker Compose-based service orchestration
- **Backup Strategy:** Automated backup scripts, scheduled execution via cron
- **Update Management:** Watchtower automated container updates with notifications
- **Documentation:** Comprehensive README, inline comments, troubleshooting guides
- **Version Control:** Git-based configuration management (planned full GitOps)

### Application Integration
- **Media Automation:** *arr ecosystem integration (Sonarr, Radarr, Prowlarr, etc.)
- **Database Administration:** PostgreSQL deployment and configuration for Nextcloud/Immich
- **Caching Layers:** Redis integration for application performance
- **API Integration:** Cloudflare API, Telegram Bot API, service webhooks
- **Smart Home:** Home Assistant device integration and automation

---

## Future Enhancements

### Security Improvements (High Priority - 1-2 weeks)

**Fail2Ban Intrusion Prevention:**
- Deploy Fail2Ban for SSH and web service protection
- Configure jails: SSH (3 attempts), HTTP 404/403 scanning detection
- Telegram integration for real-time IP ban notifications
- Custom Prometheus exporter for ban metrics
- GeoIP mapping of banned IPs for attack pattern analysis

**Network Segmentation:**
- Implement VLAN 1 (Production) and VLAN 2 (IoT/Untrusted)
- Configure inter-VLAN firewall rules (deny IoT → Production)
- Migrate smart home devices to isolated IoT network
- Document VLAN configuration in OpenWRT

**Access Hardening:**
- Enable Cloudflare Zero Trust for Heimdall dashboard
- Implement 2FA for critical services (Portainer, Grafana)
- Deploy Authelia or Authentik for unified SSO
- Audit and document all service authentication mechanisms

### Monitoring & Alerting (Medium Priority - 2-4 weeks)

**Grafana Alert Rules:**
- CPU usage > 80% sustained (5 minute window)
- Memory usage > 90% available
- Disk space > 85% full (boot drive and media drive)
- Container restart loops (3+ restarts in 10 minutes)
- Service downtime detection

**Centralized Logging:**
- Deploy Loki + Promtail stack for log aggregation
- Integrate with Grafana for unified observability
- Log retention policy (30-day rotation)
- Search and filtering capabilities

**Additional Exporters:**
- Jellyfin metrics exporter (stream count, transcode load)
- SABnzbd metrics exporter (queue depth, download rate)
- Nextcloud metrics exporter (user count, storage usage)
- UPS monitoring (when hardware acquired)

### Infrastructure Automation (Medium Priority - 1-2 months)

**Ansible Deployment:**
- Convert Docker Compose to Ansible playbooks
- Automated server provisioning from bare metal
- Idempotent configuration management
- Secret management with Ansible Vault
- Multi-environment support (dev/staging/prod)

**Backup Enhancements:**
- Offsite backup replication (cloud storage integration)
- Automated restore testing (monthly validation)
- Database-specific backup strategies (pg_dump, etc.)
- Immutable backup storage (write-once-read-many)
- Backup monitoring and alerting

**GitOps Workflow:**
- GitHub repository for infrastructure-as-code
- Automated deployment on git push (webhooks)
- Version-controlled service configurations
- Rollback capabilities
- Change tracking and audit log

### Service Expansion (Low Priority - 3-6 months)

**New Services:**
- **Vaultwarden:** Self-hosted password manager (Bitwarden alternative)
- **Paperless-ngx:** Document scanning and management
- **Audiobookshelf:** Audiobook streaming and library management
- **Actual Budget:** Personal finance and budgeting application
- **AdGuard Home:** DNS-level ad blocking and tracking prevention
- **Uptime Kuma:** Service uptime monitoring and status page

**Home Assistant Expansion:**
- Voice assistant integration (local processing)
- Energy monitoring dashboards
- Automation rule library expansion
- Mobile app presence detection
- Climate control integration

**Game Servers:**
- Additional Minecraft servers (vanilla, modded variants)
- Valheim dedicated server
- Terraria multiplayer server
- Automated backup and update scripts

### Hardware Upgrades (Long-Term - 6-12 months)

**Compute Expansion:**
- RAM upgrade: 16GB → 32GB (support more containers)
- Storage upgrade: Additional 6TB HDD (RAID 1 mirror for redundancy)
- UPS addition: Graceful shutdown on power loss, runtime monitoring
- Network upgrade: 2.5GbE NIC for faster local transfers

**High Availability:**
- Second server node (clustered deployment)
- Docker Swarm or Kubernetes evaluation
- Shared storage via NFS or GlusterFS
- Load balancing for critical services

---

## Documentation

### Repository Structure

This repository contains comprehensive documentation and configuration files:

```
homelab-infrastructure/
├─ README.md                    - This file (overview and architecture)
├─ AL-HOMELAB-COMPLETE-GUIDE.md - Detailed setup and troubleshooting guide
├─ HOMELAB-WEEKLY-TASKS.md      - Maintenance task checklist
├─ docker/
│  ├─ core/
│  │  └─ docker-compose.yml     - Management services
│  ├─ media/
│  │  └─ docker-compose.yml     - Media automation stack
│  ├─ cloud/
│  │  └─ docker-compose.yml     - Cloud services
│  └─ monitoring/
│     └─ docker-compose.yml     - Prometheus + Grafana
├─ scripts/
│  ├─ backup-homelab.sh         - Automated backup script
│  └─ homelab-status.sh         - System status checker
├─ docs/
│  ├─ network-diagram.png       - Network topology visualization
│  ├─ screenshots/              - Dashboard and UI screenshots
│  └─ guides/                   - Service-specific configuration guides
└─ LICENSE                      - MIT License
```

### Additional Documentation

**Internal Reference Guides:**
- Complete service configuration guide (API keys, credentials - separate secure document)
- Troubleshooting runbook (common issues and resolutions)
- Disaster recovery procedures (restoration from backup)
- Network diagram with IP addressing scheme

**Planned Documentation:**
- OpenWRT VLAN configuration tutorial
- Tailscale mesh network deployment guide
- Fail2Ban configuration and tuning guide
- Ansible playbook documentation
- SSL/TLS certificate automation workflow

---

## Project Timeline

**Initial Deployment:** December 2025  
**Current Phase:** Enhancement and optimization  
**Last Updated:** January 2026  
**Status:** 🟢 Active Development  

### Changelog Highlights

**January 2026:**
- Migrated from Debian 12 to Debian 13 (Trixie)
- Added Immich photo management platform
- Integrated Home Assistant for smart home control
- Deployed Nextcloud for file sync and collaboration
- Implemented automated backup system with cron scheduling
- Expanded Grafana monitoring dashboards

**December 2025:**
- Initial homelab deployment on Debian 12
- Configured 24-container Docker infrastructure
- Deployed media automation stack (*arr ecosystem)
- Integrated Tailscale mesh VPN
- Established Cloudflare Tunnel for Jellyfin public access
- Set up Prometheus + Grafana monitoring

**Planned for February 2026:**
- Fail2Ban deployment with security monitoring
- VLAN network segmentation
- Cloudflare Zero Trust for Heimdall
- Ansible-based infrastructure automation
- Expanded alerting and notification system

---

## Contact & Links

**Albert Weiner**

📧 Email: [ajgreenboy@gmail.com](mailto:ajgreenboy@gmail.com)  
💼 LinkedIn: [linkedin.com/in/al-weiner-29865529a](https://www.linkedin.com/in/al-weiner-29865529a/)  
🐙 GitHub: [github.com/ajgr33nboy](https://github.com/ajgr33nboy)  
🌐 Portfolio: [unfunky.xyz](https://unfunky.xyz)

**Live Demos:**
- 🎬 Jellyfin Media Server: [jellyfin.unfunky.xyz](https://jellyfin.unfunky.xyz)
- 📊 Monitoring Dashboard: Internal access via Tailscale VPN
- 📷 Photo Gallery: Internal access via Tailscale VPN

---

## Acknowledgments

**Technologies Used:**
- Debian Project
- Docker & Docker Compose
- Prometheus & Grafana
- Tailscale (WireGuard)
- Cloudflare
- LinuxServer.io container images
- Servarr ecosystem (*arr applications)

**Community Resources:**
- r/homelab
- r/selfhosted
- Servarr Discord community
- LinuxServer.io Discord
- Tailscale community forum

---

## License

This project documentation is released under the MIT License. Configuration files and scripts are provided as-is for educational and reference purposes.

See [LICENSE](LICENSE) file for full details.

---

**🏠 Built with passion in Minneapolis, Minnesota**  
**⚡ Powered by Debian, Docker, and caffeine**  
**🎯 Demonstrating enterprise DevOps practices at home scale**

---

*"The best way to predict the future is to build it."*
