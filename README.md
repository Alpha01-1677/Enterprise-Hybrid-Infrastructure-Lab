# Enterprise Hybrid Infrastructure & Observability Lab

A production-grade hybrid enterprise lab featuring Windows Server 2022 Active Directory Domain Services (AD DS), centralized DHCP/DNS management, Linux hardening, and containerized telemetry monitoring.

---

## 🏗️ Architecture

![Lab Architecture](docs/architecture-diagram.jpg)

* **Isolated NAT Network (`VMnet8`):** `192.168.10.0/24` with Gateway at `192.168.10.2`.
* **DC-1 (`192.168.10.10`):** Windows Server 2022 handling AD DS (`corp.local`), DNS, DHCP scope (`.50`–`.150`), and security GPOs.
* **Ubuntu Server (`192.168.10.20`):** Hardened with UFW, running Nginx (:80) and Docker Compose (Prometheus, Node Exporter, Grafana).
* **Client Workstation:** Windows 10 Enterprise joined to `corp.local` with enforced security baselines.

---

## 📋 Quick Specs & Command Matrix

| Host / Role | OS | IP Config | Management Command |
| :--- | :--- | :--- | :--- |
| **DC-1** | Windows Server 2022 | `192.168.10.10` (Static) | `dsa.msc` (AD) / `dhcpmgmt.msc` (DHCP) |
| **Ubuntu** | Ubuntu 22.04 LTS | `192.168.10.20` (Static) | `sudo ufw status` / `docker ps` |
| **Client** | Windows 10 Enterprise | Dynamic DHCP | `whoami` / `gpresult /r` |

---

## ⚡ Quick Deployment Guide

### Phase 1: Network Setup (VMware VMnet8)
1. In VMware Virtual Network Editor, set **VMnet8 (NAT)** Subnet to `192.168.10.0/24` and Gateway to `192.168.10.2`.
2. **Uncheck** "Use local DHCP service" so the Windows Domain Controller controls all leases.

### Phase 2: Windows Server 2022 Setup (`DC-1`)
1. Assign static IP `192.168.10.10`, subnet `255.255.255.0`, gateway `192.168.10.2`, DNS `127.0.0.1` (`ncpa.cpl`).
2. Install **AD DS** and **DHCP Server** via Server Manager.
3. Promote server to root domain `corp.local` and reboot.
4. Create DHCP Scope `Corp_LAN` (`192.168.10.50–150`) with Router `192.168.10.2` and DNS `192.168.10.10`. Authorize the server.
5. Create OU `Corp_Employees` ➔ `IT_Dept` and provision user `amorgan`.
6. Create & link `GPO_Workstation_Security_Baseline` enforcing a 900-second screensaver password lock.

### Phase 3: Ubuntu Hardening & Monitoring
1. Configure static IP via Netplan (`sudo nano /etc/netplan/00-installer-config.yaml`):
```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.10.20/24]
      routes:
        - to: default
          via: 192.168.10.2
      nameservers:
        addresses: [192.168.10.10, 8.8.8.8]
```
Apply using `sudo netplan apply`.

2. Install Nginx and configure the UFW firewall:
```bash
sudo apt update && sudo apt install nginx docker.io docker-compose-v2 -y
sudo systemctl enable --now nginx
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp && sudo ufw allow 80/tcp && sudo ufw allow 3000/tcp
sudo ufw enable
```

3. Start telemetry stack in `~/monitoring/docker-compose.yml`:
```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]

  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    restart: unless-stopped
    ports: ["9100:9100"]

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports: ["3000:3000"]
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
```
Launch with `docker compose up -d`.

### Phase 4: Client Join & Verification
1. Boot Windows 10 with DHCP enabled.
2. Join domain `corp.local` via `sysdm.cpl` using `CORP\Administrator`.
3. Log in as `amorgan`, change the temporary password, and verify GPO enforcement:
```cmd
whoami
gpresult /r
```
4. Access `http://192.168.10.20:3000` to verify live Grafana telemetry across subnets.

---

## 🧪 Proof of Work

### 1. Active Directory OU & User Provisioning
![Active Directory Hierarchy](docs/screenshots/01-ad-users-ou.png)

### 2. Centralized DHCP Address Leases
![DHCP Lease Scope](docs/screenshots/02-dhcp-lease.png)

### 3. GPO Baseline Verification (`gpresult /r`)
![GPO Verification](docs/screenshots/03-gpo-verification.png)

### 4. Linux Security (UFW) & Docker Containers
![Linux Security and Docker](docs/screenshots/04-linux-hardening-docker.png)

### 5. Cross-Subnet Telemetry Dashboard
![Grafana Dashboard](docs/screenshots/05-grafana-dashboard.png)

---

## 📄 Complete Project Documentation & Report

For in-depth theoretical concepts, configuration matrices, and the full step-by-step engineering walkthrough, refer to the complete technical document:

* 📥 **Download Full Lab Report:** [Enterprise-Hybrid-Infra-Lab-Report.pdf](docs/Enterprise-Hybrid-Infra-Lab-Report.pdf)
