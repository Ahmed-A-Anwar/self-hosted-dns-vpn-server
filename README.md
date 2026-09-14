# Home DNS and VPN Server
Self-hosted network-wide ad blocking, SafeSearch enforcement, and secure 
remote access — built on repurposed hardware (Intel Core 2 Duo, 2GB RAM).

## What I used
- Ubuntu Server 26.04 LTS
- Pi-hole v6 for DNS and ad blocking
- WireGuard VPN
- UFW Uncomplicated Firewall
- unattended-upgrades for automated security patching

## Setup Steps

### 1. Base OS Install
- Flashed Ubuntu Server LTS to USB using Etcher
- Installed on a legacy device
its specs : intel Core i2 Duo and 2GBs RAM
- Set static IP via netplan:
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```
then make  dhcp4: false
### 2.Pi-hole for DNS and ad blocking
to install :
```bash
curl -sSL https://install.pi-hole.net | sudo bash
```
- in Upstream DNS I chose Cloudflare with DNSSEC enabled
- Verified it is working via web UI I tried blocking TikTok domains then saw the site is inaccessible

### 3. WireGuard VPN
```bash
sudo apt install wireguard -y
```
- Generated server + client keypairs
```bash
 wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
 ```
 ```bash
 wg genkey | tee phone_private | wg pubkey > phone_public
 ```
- Configured `wg0.conf` with NAT/forwarding rules
- Verified active tunnel status via client app

### 4. SafeSearch Enforcement through DNS Override
**Goal:** Force strict SafeSearch across all devices on the network by 
resolving search engine domains to their official SafeSearch VIP addresses.

**Problem:** Pi-hole v6's `pihole-FTL` engine doesn't automatically scan 
`/etc/dnsmasq.d/` the way legacy Pi-hole did,causing custom DNS overrides 
to be silently ignored `dig` queries kept returning public Google IPs 
instead of the SafeSearch IP.

**Solution:** Applied overrides directly to `/etc/hosts`, which `pihole-FTL` 
reliably parses across versions:
```bash
echo -e "216.239.38.120 www.google.com google.com\n204.79.197.220 www.bing.com bing.com\n52.142.124.215 duckduckgo.com" | sudo tee -a /etc/hosts
sudo pihole reloaddns
```

**Verification:**
```bash
dig @127.0.0.1 www.google.com +short
```
it retured: 216.239.38.120 
so override is working
### 5. Remote Access Security Hardening
Wireguard requires an open port on the router for port forwarding, so I hardened the server 
before exposing it:

**Automated security patching:**
```bash
sudo apt update && sudo apt install -y unattended-upgrades apt-config-auto-update
sudo dpkg-reconfigure -plow unattended-upgrades
```

**Firewall zero-trust baseline i.e deny by default:**
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 51820/udp              # for Wireguard only
sudo ufw allow from 192.168.1.0/24    # trust the local network only
```

**Router port forwarding:** UDP 51820 to server's static local IP

## Results
- Network-wide Ad and tracker blocking by Pi-hole
- Enforced SafeSearch across search engines on network
- Secure remote access via WireGuard and security hardened with Uncomplicated Firewall and auto-patching
- All running on legacy hardware that would otherwise be e-waste
