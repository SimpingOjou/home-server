# Home-Server - Topolini 🐭

My personal home-server architecture running on **Ubuntu LTS** via **Docker Swarm**. Made with usability in mind. **Description is AI-generated**

## 🏗️ Architecture Overview

The project is built on a **Dual-Layer Access** model:

1. **Public Layer (Cloudflare Tunnel):** Nextcloud and Personal Site are accessible via custom domains, protected by Cloudflare's security (WAF, OTP, and Bot Protection).
2. **Private Layer (Tailscale VPN):** Sensitive management tools like Portainer and Uptime Kuma are strictly isolated within a WireGuard-based Mesh VPN.

## 🛠️ Stack & Services

| Service | Description | Access Method |
| --- | --- | --- |
| **Homepage** | Unified dashboard for all services | Public (CF Tunnel) |
| **Nextcloud** | Private cloud for file sync & collaboration | Public (CF Tunnel) |
| **Pi-hole** | Network-wide Ad-blocking & Local DNS | Local / VPN |
| **Jellyfin** | Media server for personal movies/shows | Local / VPN |
| **Uptime Kuma** | Service monitoring & Status Page | VPN Only |
| **Portainer** | Container management UI | VPN Only |

## 🔒 Security Features

* **Cloudflare Zero Trust:** Acts as a reverse proxy for public services, hiding the home IP address.
* **Tailscale Integration:** Provides an encrypted tunnel for remote management without opening ports on the router (ZTE).
* **Split DNS:** Pi-hole handles local DNS queries and blocks tracking/ads at the source.

## 🚀 Deployment

### Prerequisites

* Ubuntu Server with Docker and Swarm mode enabled.
* Tailscale installed on the host.
* Cloudflare `cloudflared` token.

### Setup

1. Clone the repository:
```bash
git clone https://github.com/SimpingOjou/home-server.git
cd home-server
```


2. Create and populate your secrets:
```bash
cp .env.example .env
vim .env
```


3. Deploy the stack:
```bash
export $(grep -v '^#' .env | xargs)
sudo -E docker stack deploy -c docker-compose.yml main
```

## 🌐 Networking 

In this setup, Ubuntu acts as a **Layer 4 Traffic Controller**. The security is based on a "Default Deny" policy where only specific tunnels and local interfaces are trusted.

### 1. The Entrance: What gets in?

The networking is split into three distinct paths, each with a different trust level:

* **The Public Tunnel (Cloudflare):**
* **Traffic:** Only HTTPS (port 443) traffic for defined hostnames (e.g., `cloud.topolini.cc`) is routed back through the tunnel.
* **Security:** Ubuntu's firewall (UFW) doesn't even see these ports as "open" because the connection is initiated from the inside.


* **The Virtual Private Bridge (Tailscale):**
* **Mechanism:** A virtual network interface (`tailscale0`) is created.
* **Traffic:** All traffic is encrypted via WireGuard.
* **Trust:** This is our "Management Lane". Portainer, Uptime Kuma, and SSH are reachable only through the 100.x.x.x IP range.


* **The Local LAN:**
* **Mechanism:** The physical Ethernet/Wi-Fi interface.
* **Security:** Protected by the router's firewall and UFW. Only DNS (53) and Jellyfin (8096) are exposed to the local 192.168.x.x network.
* 

### 2. Docker Networking:

Docker doesn't use the standard Ubuntu networking directly; it creates its own subnet:

* **Bridge Mode (`lab_network`):** Containers live in an isolated subnet (e.g., `172.18.0.x`). They can talk to each other by name (DNS resolution via Docker), but they are invisible to the outside world unless a port is "published".
* **Host Mode:** Used for **Pi-hole (DNS)** and **Jellyfin**. This removes the Docker NAT overhead, allowing the service to "bind" directly to the laptop's IP.
* There is a need to manually open UFW ports—Host Mode bypasses Docker's internal firewall management.



### 3. Firewall Rules

When **Uptime Kuma** (inside Docker) tries to monitor **Pi-hole** (on the Host), the traffic follows this path:

1. **Container** -> **Docker Gateway** (`172.18.0.1`) -> **Host Loopback**.
2. **UFW** sees a packet coming from a "strange" 172.x.x.x IP and, by default, drops it.
3. **The Fix:** We added explicit rules to trust the Docker subnet:
`sudo ufw allow from 172.16.0.0/12 to any port 53 proto udp`

### 🛠️ Useful commands

| Command | Purpose |
| --- | --- |
| `sudo ss -tulpn` | See exactly which service is listening on which port. |
| `sudo ufw status numbered` | Check the firewall rules and their order. |
| `ip addr show` | List all network interfaces (eth0, tailscale0, docker0). |
| `docker network inspect lab_network` | See the internal IPs assigned to your containers. |

---
