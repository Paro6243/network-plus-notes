# Lab 03 — DHCP Analysis & IP Configuration

**Date:** May 17, 2026
**Tool:** ip, nmcli, Terminal
**OS:** Parrot OS
**Difficulty:** Beginner
**Topic:** DHCP, IP Assignment, Lease Info, Routing Table
**Reference:** Professor Messer N+ — DHCP

---

## 🎯 Objective

Analyze how DHCP works in practice by inspecting the IP address, gateway, DNS server, and lease information automatically assigned to my machine by a DHCP server.

---

## 📖 Key Concepts (From Prof Messer)

### What is DHCP?
DHCP (Dynamic Host Configuration Protocol) automatically assigns IP configuration to devices on a network. Without it, every device would need to be manually configured.

### What DHCP Assigns (DORA Process)

```
Device                          DHCP Server (Router/Hotspot)
  │                                      │
  │──── DISCOVER ───────────────────────▶│  "Anyone have an IP for me?"
  │                                      │
  │◀─── OFFER ──────────────────────────│  "I'll give you 10.155.101.248"
  │                                      │
  │──── REQUEST ────────────────────────▶│  "Yes I'll take that IP"
  │                                      │
  │◀─── ACKNOWLEDGE ────────────────────│  "It's yours! Lease = 3121 sec"
```

### What DHCP Provides
- IP Address
- Subnet Mask
- Default Gateway
- DNS Server
- Lease Time

---

## 🔬 Lab Commands & Real Output

### 1. View Network Interfaces & DHCP-Assigned IP
```bash
$ ip addr show

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host

2: eno1: <NO-CARRIER,BROADCAST,MULTICAST,UP> state DOWN
    link/ether 04:0e:3c:04:6a:26

3: wlo1: <BROADCAST,MULTICAST,UP,LOWER_UP> state UP
    link/ether 80:91:33:af:70:95
    inet 10.155.101.248/24 brd 10.155.101.255 scope global dynamic wlo1
    valid_lft 3121sec preferred_lft 3121sec
    inet6 2401:4900:c006:26be:5e31:781e:bcd8:a1b6/64 scope global dynamic
    inet6 fe80::2fe9:5011:6738:36ed/64 scope link
```

**What I observed:**
- `lo` = loopback interface, always `127.0.0.1`, used for internal communication
- `eno1` = ethernet port, state **DOWN** (cable not connected)
- `wlo1` = WiFi interface, state **UP** (connected)
- DHCP assigned `10.155.101.248/24` to my WiFi interface
- Lease time = **3121 seconds** (~52 mins) then DHCP renews
- Both IPv4 and IPv6 assigned automatically by DHCP

---

### 2. View Routing Table (Gateway assigned by DHCP)
```bash
$ ip route show

default via 10.155.101.43 dev wlo1 proto dhcp src 10.155.101.248 metric 600
10.155.101.0/24 dev wlo1 proto kernel scope link src 10.155.101.248 metric 600
```

**What I observed:**
- Default gateway = `10.155.101.43` — set automatically by DHCP
- `proto dhcp` confirms this route was assigned by DHCP, not manually
- My machine's subnet = `10.155.101.0/24` (IPs .1 to .254 on this network)
- `metric 600` = route priority value

---

### 3. Full DHCP Lease Details (nmcli)
```bash
$ nmcli device show wlo1

GENERAL.DEVICE:      wlo1
GENERAL.TYPE:        wifi
GENERAL.HWADDR:      80:91:33:AF:70:95
GENERAL.MTU:         1500
GENERAL.STATE:       100 (connected)
GENERAL.CONNECTION:  iQOO Neo7 Pro

IP4.ADDRESS[1]:      10.155.101.248/24
IP4.GATEWAY:         10.155.101.43
IP4.DNS[1]:          10.155.101.43

IP6.ADDRESS[1]:      2401:4900:c006:26be:5e31:781e:bcd8:a1b6/64
IP6.ADDRESS[2]:      fe80::2fe9:5011:6738:36ed/64
IP6.GATEWAY:         fe80::90d6:3fff:fef4:9c29
IP6.DNS[1]:          2401:4900:c006:26be::57
```

**What I observed:**
- DHCP server = my **phone hotspot (iQOO Neo7 Pro)**
- Phone is acting as a mini DHCP server — real world example!
- MAC address `80:91:33:AF:70:95` is how DHCP identifies my device
- DNS server = same IP as gateway (`10.155.101.43`) — phone handles both
- MTU = `1500` bytes — standard frame size
- Full IPv6 configuration also assigned automatically

---

## 📊 Full DHCP Assignment Summary

| Parameter | Value | Assigned By |
|-----------|-------|------------|
| IP Address | `10.155.101.248` | DHCP (phone hotspot) |
| Subnet Mask | `/24` (255.255.255.0) | DHCP |
| Default Gateway | `10.155.101.43` | DHCP |
| DNS Server | `10.155.101.43` | DHCP |
| Lease Time | 3121 seconds (~52 min) | DHCP |
| IPv6 Address | `2401:4900:c006:26be:.../64` | DHCP |
| MAC Address | `80:91:33:AF:70:95` | Hardware (used by DHCP to identify device) |

---

## ⚠️ Troubleshooting Note — Why Some Commands Failed

During this lab, older DHCP commands failed:
```bash
cat /var/lib/dhcp/dhclient.leases     # ❌ File not found
sudo journalctl | grep -i dhcp        # ❌ No permission
```

**Why:** Parrot OS uses **NetworkManager** to manage network connections — not the older `dhclient` program. NetworkManager stores lease info differently and uses its own tools.

**Fix:** Always use `nmcli` on NetworkManager-based systems:
```bash
nmcli device show wlo1    # ✅ Correct tool for this OS
```

**The lesson:** In real IT work, knowing *which* network manager your OS uses is critical. Always match your tools to your system.

---

## 💡 Key Takeaways

1. **DHCP uses the DORA process** — Discover, Offer, Request, Acknowledge
2. **MAC address is how DHCP identifies devices** — not hostname
3. **Gateway and DNS are both assigned by DHCP** — not just the IP
4. **Lease time matters** — when it expires, DHCP renews the IP (may change!)
5. **Phone hotspots run DHCP servers** — real world mini DHCP in action
6. **NetworkManager = use nmcli** — not older dhclient commands on modern Linux
7. **`ip route show`** reveals what DHCP set as your default gateway

---

## 🔗 Related Topics

- CompTIA N+ Domain 1: Networking Concepts
- DHCP Relay Agent — for DHCP across subnets
- APIPA — what happens when DHCP fails (169.254.x.x)
- Professor Messer N+ Section: DHCP

---

*Lab performed on Parrot OS | Part of Network+ study journey | [github.com/Paro6243](https://github.com/Paro6243)*
