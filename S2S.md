# pfSense Site-to-Site IPsec VPN Configuration

## Network Overview

| | Site A | Site B |
|---|---|---|
| **VPCs** | 10.113.0.0/16, 10.114.0.0/16 | 10.115.0.0/16 |
| **pfSense IP** | in 10.114.0.0/24 | in 10.115.4.0/24 |
| **WAN IP** | `<WAN_A>` | `<WAN_B>` |

> Replace `<WAN_A>` and `<WAN_B>` with the actual public/WAN IPs of each pfSense box.

---

## Step 1 — Configure Phase 1 (IKE) on Site A

1. Navigate to **VPN → IPsec → Tunnels**
2. Click **+ Add P1**
3. Fill in:

| Field | Value |
|---|---|
| Key Exchange version | **IKEv2** |
| Internet Protocol | **IPv4** |
| Interface | **WAN** |
| Remote Gateway | `<WAN_B>` (public IP of Site B pfSense) |
| Description | `Site-A-to-Site-B` |
| **Authentication Method** | **Mutual PSK** |
| My identifier | **My IP address** |
| Peer identifier | **Peer IP address** |
| Pre-Shared Key | *(generate a strong shared secret, e.g. 32+ chars)* |
| **Phase 1 Proposal (Encryption)** | |
| Encryption Algorithm | **AES 256-GCM** with 128-bit key length |
| Hash Algorithm | **SHA256** |
| DH Group | **14 (2048 bit)** or **19 (ECP 256)** |
| Lifetime | **28800** |

4. Click **Save**

---

## Step 2 — Configure Phase 2 (IPsec SA) on Site A

Since Site A has **2 VPCs** and Site B has **1 VPC**, you need **2 Phase 2 entries** on each side.

### Phase 2 Entry #1: 10.113.0.0/16 ↔ 10.115.0.0/16

1. Under the Phase 1 you just created, click **+ Add P2**
2. Fill in:

| Field | Value |
|---|---|
| Mode | **Tunnel IPv4** |
| Local Network | **Network** → `10.113.0.0/16` |
| Remote Network | **Network** → `10.115.0.0/16` |
| Description | `SiteA-VPC1-to-SiteB` |
| **Phase 2 Proposal** | |
| Encryption Algorithm | **AES 256-GCM** |
| Hash Algorithm | **SHA256** |
| PFS key group | **14 (2048 bit)** |
| Lifetime | **3600** |

3. Click **Save**

### Phase 2 Entry #2: 10.114.0.0/16 ↔ 10.115.0.0/16

1. Click **+ Add P2** again under the same Phase 1
2. Fill in:

| Field | Value |
|---|---|
| Mode | **Tunnel IPv4** |
| Local Network | **Network** → `10.114.0.0/16` |
| Remote Network | **Network** → `10.115.0.0/16` |
| Description | `SiteA-VPC2-to-SiteB` |
| *(Phase 2 Proposal)* | Same as Entry #1 |

3. Click **Save**
4. Click **Apply Changes** at the top

---

## Step 3 — Configure Phase 1 (IKE) on Site B

1. On **Site B pfSense**, go to **VPN → IPsec → Tunnels**
2. Click **+ Add P1**
3. Fill in — **mirror** of Site A:

| Field | Value |
|---|---|
| Key Exchange version | **IKEv2** |
| Remote Gateway | `<WAN_A>` (public IP of Site A) |
| Pre-Shared Key | *(same PSK as Site A)* |
| Description | `Site-B-to-Site-A` |
| *(all other settings)* | **Identical** to Site A Phase 1 |

4. Click **Save**

---

## Step 4 — Configure Phase 2 on Site B

### Phase 2 Entry #1: 10.115.0.0/16 ↔ 10.113.0.0/16

| Field | Value |
|---|---|
| Local Network | **Network** → `10.115.0.0/16` |
| Remote Network | **Network** → `10.113.0.0/16` |

### Phase 2 Entry #2: 10.115.0.0/16 ↔ 10.114.0.0/16

| Field | Value |
|---|---|
| Local Network | **Network** → `10.115.0.0/16` |
| Remote Network | **Network** → `10.114.0.0/16` |

> Phase 2 proposals must match Site A exactly. Click **Apply Changes**.

---

## Step 5 — Firewall Rules on Both Sites

### 5a. WAN Rule (allow IPsec traffic)

Usually pfSense auto-allows IPsec on WAN. Verify at **Firewall → Rules → WAN**:

| Field | Value |
|---|---|
| Action | **Pass** |
| Protocol | **UDP** |
| Destination Port | **500** (IKE) and **4500** (NAT-T) |
| Source | Remote pfSense WAN IP |

Also ensure **ESP** (protocol 50) is allowed.

### 5b. IPsec Interface Rules (allow traffic through the tunnel)

Go to **Firewall → Rules → IPsec**:

**On Site A:**

| Field | Value |
|---|---|
| Action | **Pass** |
| Source | `10.115.0.0/16` |
| Destination | `10.113.0.0/16, 10.114.0.0/16` *(or use **any** to start)* |
| Protocol | **Any** |

**On Site B:**

| Field | Value |
|---|---|
| Action | **Pass** |
| Source | `10.113.0.0/16, 10.114.0.0/16` |
| Destination | `10.115.0.0/16` |
| Protocol | **Any** |

> You can tighten these rules later to specific protocols/ports.

---

## Step 6 — Enable IPsec & Bring Up the Tunnel

1. Go to **VPN → IPsec** → check **Enable IPsec** at the top → **Save**
2. Go to **Status → IPsec** → click **Connect P1 and P2s** (🔌 icon)
3. Both Phase 1 and Phase 2 entries should show **ESTABLISHED**

---

## Step 7 — Add Static Routes (if needed)

If your pfSense on Site A is in `10.114.0.0/24` but needs to route traffic for `10.113.0.0/16`, you may need:

- **System → Routing → Static Routes** on Site A pfSense:
  - Destination: `10.113.0.0/16`
  - Gateway: the gateway in the 10.114.0.0/24 subnet that can reach the 10.113.0.0/16 VPC

This depends on your cloud/network setup (e.g., VPC peering, shared gateway, etc.).

---

## Step 8 — Verify

1. **Status → IPsec** — both P1 and all P2 SAs should be **Established**
2. From a host in `10.115.0.0/16`, ping a host in `10.113.0.0/16` and `10.114.0.0/16`
3. From Site A, ping a host in `10.115.0.0/16`
4. If it fails, check **Status → System Logs → IPsec** for mismatch errors

---

## Common Troubleshooting

| Symptom | Check |
|---|---|
| Phase 1 won't establish | PSK mismatch, encryption/DH mismatch, WAN firewall blocking UDP 500/4500 |
| Phase 1 OK but Phase 2 fails | Local/Remote network mismatch between sites, encryption mismatch |
| Tunnel up but no traffic | Missing IPsec firewall rule, missing static route, or asymmetric routing |
| Intermittent drops | DPD (Dead Peer Detection) settings, lifetime mismatch |
