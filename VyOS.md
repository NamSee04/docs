# VyOS Site-to-Site IPsec VPN Configuration

## Network Overview

| | Site A | Site B |
|---|---|---|
| **VPCs** | 10.113.0.0/16, 10.114.0.0/16 | 10.115.0.0/16 |
| **VyOS Private IP** | 10.114.0.3 | 10.115.4.254 |
| **WAN IP** | 103.245.254.51 | 183.91.8.151 |

---

## Site A Configuration (103.245.254.51)

```bash
configure

# --- IKE (Phase 1) ---
set vpn ipsec ike-group IKE-SITE mode ikev2
set vpn ipsec ike-group IKE-SITE proposal 1 dh-group 14
set vpn ipsec ike-group IKE-SITE proposal 1 encryption aes256
set vpn ipsec ike-group IKE-SITE proposal 1 hash sha256
set vpn ipsec ike-group IKE-SITE dead-peer-detection action restart
set vpn ipsec ike-group IKE-SITE dead-peer-detection interval 30
set vpn ipsec ike-group IKE-SITE dead-peer-detection timeout 120
set vpn ipsec ike-group IKE-SITE key-exchange ikev2

# --- ESP (Phase 2) ---
set vpn ipsec esp-group ESP-SITE proposal 1 encryption aes256
set vpn ipsec esp-group ESP-SITE proposal 1 hash sha256
set vpn ipsec esp-group ESP-SITE pfs dh-group14

# --- Interface binding ---
set vpn ipsec interface eth0

# --- Site-to-Site Peer (Site B) ---
set vpn ipsec site-to-site peer SITE-B authentication mode pre-shared-secret
set vpn ipsec site-to-site peer SITE-B authentication pre-shared-secret 'YOUR_STRONG_PSK_HERE'
set vpn ipsec site-to-site peer SITE-B remote-address 183.91.8.151
set vpn ipsec site-to-site peer SITE-B local-address 103.245.254.51
set vpn ipsec site-to-site peer SITE-B ike-group IKE-SITE
set vpn ipsec site-to-site peer SITE-B default-esp-group ESP-SITE

# --- Tunnel 0: 10.113.0.0/16 <-> 10.115.0.0/16 ---
set vpn ipsec site-to-site peer SITE-B tunnel 0 local prefix 10.113.0.0/16
set vpn ipsec site-to-site peer SITE-B tunnel 0 remote prefix 10.115.0.0/16

# --- Tunnel 1: 10.114.0.0/16 <-> 10.115.0.0/16 ---
set vpn ipsec site-to-site peer SITE-B tunnel 1 local prefix 10.114.0.0/16
set vpn ipsec site-to-site peer SITE-B tunnel 1 remote prefix 10.115.0.0/16

commit
save
