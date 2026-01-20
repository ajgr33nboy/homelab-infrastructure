# Homelab Infrastructure

**Production-grade home server environment with enterprise security practices**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker)](https://www.docker.com/)
[![Linux](https://img.shields.io/badge/OS-Debian-A81D33?logo=debian)](https://www.debian.org/)
[![Network](https://img.shields.io/badge/Router-OpenWRT-00B5E2?logo=openwrt)](https://openwrt.org/)

---

## Table of Contents

- [Overview](#overview)
- [Network Architecture](#network-architecture)
- [Infrastructure Components](#infrastructure-components)
- [Security Implementation](#security-implementation)
- [Services & Applications](#services--applications)
- [Monitoring & Observability](#monitoring--observability)
- [Remote Access](#remote-access)
- [Hardware Specifications](#hardware-specifications)
- [Documentation](#documentation)

---

## Overview

This repository documents a production homelab environment designed with enterprise-grade security practices and modern infrastructure patterns. The infrastructure emphasizes defense-in-depth security, network segmentation, automated monitoring, and containerized service deployment.

**Key Principles:**
- Defense in Depth: Multi-layer security architecture
- Network Segmentation: VLAN isolation for trusted and untrusted devices
- Zero Trust: Deny-by-default posture with granular access control
- Observability: Comprehensive monitoring and alerting infrastructure
- Infrastructure as Code: Containerized services with declarative configuration

**External Access:**
- Dashboard: [home.unfunky.xyz](https://home.unfunky.xyz) (Cloudflare Zero Trust protected)
- Monitoring: [grafana.unfunky.xyz](https://grafana.unfunky.xyz) (GitHub OAuth authentication)

---

## Network Architecture

![Network Diagram](network-diagram.png)

### Topology Overview

```
Internet (Xfinity ISP / Dynamic IP with DDNS)
    ↓
Cloudflare (DNS Management + Zero Trust Gateway + DDoS Protection)
    ↓
ASUS RT-AC3100 Router (OpenWRT, VLAN Controller, Dual-band WiFi)
    ↓
├─ VLAN 1 (Production) ──→ SFF Server, Raspberry Pi, Trusted Devices
└─ VLAN 2 (IoT)        ──→ Smart Home Devices (Network Isolated)
    ↓
Docker Services (Nginx Proxy Manager, Media Stack, Monitoring)
    ↓
Tailscale Mesh VPN (Encrypted Remote Access)
```

### Design Rationale

| Component | Technology Choice | Justification |
|-----------|------------------|---------------|
| **Network Segmentation** | VLAN-based isolation | Prevents lateral movement from compromised IoT devices to production infrastructure |
| **Router Firmware** | OpenWRT | Granular control over routing, VLANs, firewall rules; superior to stock firmware |
| **VPN Architecture** | Tailscale WireGuard mesh | Zero-configuration, no port forwarding required, encrypted peer-to-peer connections |
| **Service Deployment** | Docker containerization | Simplified dependency management, version control, rapid rollback capability |
| **SSH Authentication** | Key-based only | Eliminates password-based brute-force attack vector entirely |

---

## Infrastructure Components

### Network Gateway

**ASUS RT-AC3100 (OpenWRT 23.05)**
- VLAN tagging and inter-VLAN firewall enforcement
- Dual-band WiFi (2.4GHz + 5GHz) with SSID-to-VLAN mapping
- MAC address filtering with deny-by-default posture
- DHCP reservations for static IP assignment
- Hardware: Broadcom BCM4709C0 (1.4GHz dual-core), 4x Gigabit LAN ports

### Physical Hosts

#### Primary Server: Custom SFF x86 System

| Component | Specification |
|-----------|---------------|
| **Motherboard** | MSI B450 |
| **Processor** | AMD Ryzen 5 2600 (6 cores, 12 threads) |
| **Memory** | 16GB DDR4 |
| **Storage** | 120GB NVMe (system), 6TB HDD (media library) |
| **GPU** | Dedicated GPU for hardware-accelerated transcoding |
| **Operating System** | Debian 12 (Bookworm) |
| **Network** | Gigabit Ethernet (VLAN 1 - Production) |
| **Power** | ~80W idle, ~150W under load |
| **Uptime** | 99.5%+ (Grafana monitored) |

**Storage Layout:**
```
/dev/nvme0n1 (120GB NVMe) - Root filesystem, Docker volumes
/dev/sda1    (6TB HDD)    - Media library (Plex/Jellyfin content)
```

**Network Configuration:**
```
eth0:        192.168.1.x/24  (Production VLAN)
tailscale0:  100.x.x.x/32    (Mesh VPN interface)
```

**Build Notes:** System underwent complete hardware migration including motherboard replacement, NVMe storage integration, and GPU installation for Plex/Jellyfin hardware transcoding offload.

#### Secondary Host: Raspberry Pi 3B+

- **Purpose:** Moode Audio Server for networked streaming to legacy audio equipment
- **Network:** Wired Ethernet on Production VLAN
- **Function:** Dedicated audio streaming appliance

---

## Security Implementation

### Network Security Architecture

**Firewall Policy:**
```
Default Policy:        DENY ALL
Inbound Allowed:       SSH (key-only, local network), HTTP/HTTPS (via reverse proxy)
Outbound Allowed:      Whitelisted destinations only
Inter-VLAN Traffic:    Production ↔ IoT communication blocked bidirectionally
```

**Access Control Mechanisms:**
- MAC address filtering: Deny-by-default whitelist on both VLANs
- SSH hardening:
  - Password authentication globally disabled
  - Root login disabled
  - RSA 4096-bit key-based authentication only
  - Fail2Ban intrusion prevention (see Monitoring section)
- Cloudflare Zero Trust: Application-layer authentication for public endpoints
- Nginx Proxy Manager: Centralized reverse proxy with automatic SSL/TLS certificate management

### Intrusion Prevention & Detection

**Fail2Ban Implementation:**
- Protected services: SSH (port 22), Nginx Proxy Manager (ports 80, 443)
- Ban duration: 24 hours (86400 seconds)
- Detection thresholds: SSH (3 attempts), NPM (10 attempts)
- Detection window: 10 minutes (600 seconds)
- Monitored patterns: HTTP 404/403/401/400 status codes (vulnerability scanning detection)
- Alert mechanism: Real-time Telegram notifications on IP ban/unban events

**Custom Security Exporters:**
- Fail2Ban Prometheus exporter (Python): Exposes ban metrics for time-series analysis
- GeoIP exporter (Python): Maps banned IPs to geographic locations for attack pattern analysis

### Attack Surface Reduction

| Threat Vector | Mitigation Strategy |
|--------------|---------------------|
| **Public SSH exposure** | SSH accessible only via Tailscale VPN or local network; not exposed to internet |
| **Open ports** | Only ports 80/443/25565 exposed; all HTTP traffic proxied through Cloudflare |
| **IoT device compromise** | Complete VLAN isolation; IoT devices cannot access production network |
| **Credential attacks** | Password authentication eliminated; key-based authentication mandatory |

---

## Services & Applications

All services deployed as Docker containers, managed via Docker Compose and orchestrated through Portainer.

### Media Automation Stack

| Service | Function | Network Access |
|---------|----------|----------------|
| **Plex** | Media streaming server with hardware transcoding | Internal + Tailscale VPN |
| **Jellyfin** | Open-source media server (Plex alternative) | Internal + Tailscale VPN |
| **Sonarr** | TV series acquisition and library management | Internal only |
| **Radarr** | Movie acquisition and library management | Internal only |
| **Lidarr** | Music acquisition and library management | Internal only |
| **Prowlarr** | Indexer manager for *arr ecosystem | Internal only |
| **SABnzbd** | Usenet download client | Internal only |
| **Overseerr** | Media request and discovery interface | Internal + Tailscale VPN |

### Infrastructure & Management Services

| Service | Function | Network Access |
|---------|----------|----------------|
| **Nginx Proxy Manager** | Reverse proxy with SSL certificate automation | Internal only |
| **Heimdall** | Application dashboard and service directory | **Public** (home.unfunky.xyz) |
| **Portainer** | Docker container orchestration UI | Internal only |
| **Prometheus** | Time-series metrics database | Internal only |
| **Grafana** | Metrics visualization and alerting platform | **Public** (grafana.unfunky.xyz) |
| **cAdvisor** | Container-level resource monitoring | Internal only |

### Game Servers

| Service | Function | Network Access |
|---------|----------|----------------|
| **Minecraft Server** | Modded Forge multiplayer server | Port-forwarded (25565) |

### Configuration Management

- Docker Compose: Declarative service definitions with version control
- Portainer Stacks: Visual stack management and deployment
- YAML configuration: All service configs stored in version-controlled repository

---

## Monitoring & Observability

### Monitoring Infrastructure

**Core Components:**
- **Prometheus:** Time-series metrics database (15-second scrape interval)
- **Grafana:** Visualization platform with real-time dashboards
- **Node Exporter:** System-level metrics (CPU, RAM, disk, network)
- **cAdvisor:** Container-level resource tracking
- **Custom Exporters:** Fail2Ban metrics, GeoIP attack mapping

**Deployment:** All monitoring components run as Docker containers in dedicated Portainer stack

### Grafana Dashboards

**1. System Overview (Node Exporter Full - Dashboard ID 1860)**
- CPU utilization (per-core and aggregate)
- Memory usage and available capacity
- Disk I/O and filesystem utilization
- Network throughput and bandwidth
- System uptime and load averages

**2. Docker Container Metrics (cAdvisor - Dashboard ID 193)**
- Per-container CPU consumption
- Per-container memory allocation and usage
- Container network I/O statistics
- Container filesystem usage
- Container health status and restart count
- Monitoring coverage: 12+ active containers

**3. Security Monitoring (Custom - Fail2Ban + GeoIP)**
- Currently banned IP addresses (per jail)
- Total ban count over time (cumulative)
- Failed authentication attempts timeline
- Geographic attack distribution (world map visualization)
- Real-time security event metrics

### Data Collection Pipeline

```
Docker Containers → cAdvisor (port 8081) ──┐
System Metrics → Node Exporter (port 9100) ─┤
Fail2Ban → Custom Python Exporter (port 9191) ─┼→ Prometheus (port 9090) → Grafana (port 3000)
Attack GeoIP → Custom Python Exporter (port 9192) ─┘
```

### Alert & Notification System

**Telegram Bot Integration:**
- Bot: Homelab Security Bot
- Real-time notifications for:
  - IP ban events (includes jail name, IP address, failure count)
  - IP unban events
  - Jail start/stop events
- Future planned alerts: Disk usage warnings, CPU/memory thresholds, container failures

### Data Protection & Backup Strategy

**Tier 1: System Snapshots**
- Full filesystem snapshots for rapid recovery
- Recovery Time Objective (RTO): <15 minutes for critical services

**Tier 2: Configuration Synchronization**
- Syncthing-based automated replication
- Docker Compose files, scripts, and critical configurations
- Multi-device sync: Server ↔ Laptop

**Tier 3: Off-site Backup (Planned)**
- External cloud storage integration
- Full system restore capability: <1 hour (bare metal reinstall + restore)

---

## Remote Access

### Tailscale VPN Mesh Network

**Architecture:** WireGuard-based peer-to-peer encrypted mesh network

**Connected Nodes:**
- SFF Debian Server (designated exit node)
- Personal laptop
- Mobile device
- Trusted family devices (limited access)

**Advantages:**
- No central VPN server bottleneck
- Automatic NAT traversal (eliminates port forwarding)
- Sub-50ms latency for remote administration
- Direct peer-to-peer connections when network topology permits
- Automatic cryptographic key rotation

**Use Cases:**
- Secure SSH access to server infrastructure
- Access to internal web services (Plex, Portainer, Sonarr/Radarr, etc.)
- Network-level access to Production VLAN from remote locations

### Public Web Access

**Grafana Monitoring Dashboard:**
- URL: [https://grafana.unfunky.xyz](https://grafana.unfunky.xyz)
- Authentication: Cloudflare Zero Trust with GitHub OAuth
- SSL/TLS: Let's Encrypt certificates via Nginx Proxy Manager
- Security layers: Cloudflare DDoS protection → Zero Trust authentication → NPM reverse proxy → Grafana authentication

**Heimdall Application Dashboard:**
- URL: [https://home.unfunky.xyz](https://home.unfunky.xyz)
- Authentication: Cloudflare Zero Trust
- Function: Central portal for accessing internal services

---

## Hardware Specifications

### Server Hardware Detail

**Form Factor:** Small Form Factor (SFF) for space-efficient deployment

**Thermal Management:**
- Aftermarket CPU cooler
- Optimized case airflow with multiple intake/exhaust fans
- Idle temperature: ~45°C, Load temperature: ~65°C

**Power Characteristics:**
- Idle consumption: ~80W
- Peak load: ~150W
- Annual estimated cost: ~$105 @ $0.15/kWh

**Reliability:**
- Measured uptime: 99.5%+ (Grafana monitoring)
- Unplanned downtime: <4 hours/month (primarily maintenance windows)

### Network Equipment Detail

**ASUS RT-AC3100**
- Chipset: Broadcom BCM4709C0 (1.4GHz dual-core ARM)
- WiFi capabilities: AC3100 class (2167 Mbps @ 5GHz, 1000 Mbps @ 2.4GHz)
- Ethernet: 4x Gigabit LAN, 1x Gigabit WAN
- Firmware: OpenWRT 23.05 (custom build)
- Memory: 512MB RAM, 128MB NAND flash

---

## Documentation

### Repository Structure

This repository contains comprehensive documentation for the entire homelab infrastructure:

- `SECURITY-MONITORING.md`: Detailed Fail2Ban implementation, configuration, and security metrics
- `network-diagram.png`: Visual network topology and architecture diagram
- `/docs/images/`: Screenshots of dashboards, monitoring interfaces, and system configurations

### Planned Documentation

- OpenWRT VLAN configuration guide
- Docker Compose service templates and deployment procedures
- Tailscale mesh network setup and configuration
- Disaster recovery runbook and restoration procedures
- SSL/TLS certificate management workflows

---

## Future Enhancements

### Short-Term Objectives (1-2 weeks)
- Implement Grafana alerting rules for system thresholds (disk, CPU, memory)
- Expand backup automation with scheduled off-site replication
- Add security audit logging (Lynis, ClamAV integration)
- Document current configuration in infrastructure-as-code format

### Medium-Term Objectives (1-2 months)
- Migrate to Ansible-based infrastructure provisioning
- Implement centralized logging (Loki + Promtail stack)
- Add UPS with automated graceful shutdown scripts
- Enhance SSL/TLS automation with scheduled certificate renewal monitoring

### Long-Term Objectives (3-6 months)
- Evaluate multi-node orchestration (Docker Swarm or Kubernetes)
- Implement GitOps workflow for service deployments
- Deploy network intrusion detection system (Suricata/Snort)
- Create custom status dashboard with real-time infrastructure health

---

## Technical Skills Demonstrated

This homelab infrastructure demonstrates proficiency in:

**Networking:**
- VLAN configuration and network segmentation
- Firewall rule design and implementation
- VPN architecture (WireGuard/Tailscale)
- Reverse proxy and SSL/TLS certificate management

**Systems Administration:**
- Linux server administration (Debian)
- Docker containerization and orchestration
- Service deployment and lifecycle management
- System monitoring and performance optimization

**Security:**
- Defense-in-depth architecture design
- Intrusion prevention system implementation (Fail2Ban)
- Zero-trust access control
- SSH hardening and key-based authentication

**Observability:**
- Metrics collection and time-series databases (Prometheus)
- Dashboard design and visualization (Grafana)
- Custom metrics exporter development (Python)
- Alert design and notification systems

**Infrastructure as Code:**
- Declarative configuration management (Docker Compose, YAML)
- Version-controlled infrastructure definitions
- Automated deployment workflows

---

## Contact Information

**Albert Weiner**

Email: ajgreenboy@gmail.com  
LinkedIn: [linkedin.com/in/al-weiner-29865529a](https://www.linkedin.com/in/al-weiner-29865529a/)  
GitHub: [github.com/ajgr33nboy](https://github.com/ajgr33nboy)  
Portfolio: [unfunky.xyz](https://unfunky.xyz)

---

## License

This project documentation is released under the MIT License. See [LICENSE](LICENSE) file for details.

---

**Project Status:** Active Development  
**Last Updated:** January 2026  
**Location:** Minneapolis, Minnesota, USA