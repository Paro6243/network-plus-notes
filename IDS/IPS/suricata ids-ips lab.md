# Suricata IDS/IPS Lab on Parrot OS

## Overview
This lab demonstrates setting up Suricata as both an **Intrusion Detection System (IDS)** and an **Intrusion Prevention System (IPS)** on a single machine using the loopback interface.

| Detail | Value |
|--------|-------|
| OS | Parrot OS (lory) |
| Tool | Suricata 6.0.10 |
| Interface | Loopback (`lo`) |
| Mode Tested | IDS (passive) → IPS (inline/blocking) |

---

## Table of Contents
- [Installation](#installation)
- [Configuration](#configuration)
- [Phase 1 — IDS Mode](#phase-1--ids-mode)
- [Phase 2 — IPS Mode](#phase-2--ips-mode)
- [Cleanup](#cleanup)
- [Summary](#summary)

---

## Installation

Suricata requires `libhtp2` as a dependency. On Parrot OS lory, this package was missing from the repo and had to be manually downloaded from Debian.

```bash
# Download missing dependency from Debian repos (Parrot OS is Debian 12 based)
wget http://ftp.debian.org/debian/pool/main/libh/libhtp/libhtp2_0.5.42-1+deb12u1_amd64.deb
sudo dpkg -i libhtp2_0.5.42-1+deb12u1_amd64.deb

# Install Suricata
sudo apt install suricata --fix-missing -y

# Fix any broken dependencies
sudo apt --fix-broken install

# Verify installation
suricata --version
```

### Fix GPG Key Error
If you see `NO_PUBKEY 7A8286AF0E81EE4A` during apt update:

```bash
sudo gpg --keyserver keyserver.ubuntu.com --recv-keys 7A8286AF0E81EE4A
sudo gpg --export 7A8286AF0E81EE4A | sudo tee /etc/apt/trusted.gpg.d/parrot-archive-keyring.gpg > /dev/null
sudo apt update
```

---

## Configuration

### 1. Update Rules
```bash
sudo suricata-update
```
Rules are saved to `/var/lib/suricata/rules/suricata.rules`.

### 2. Fix Rule Path in Config
Edit `/etc/suricata/suricata.yaml` and update the default rule path:

```yaml
default-rule-path: /var/lib/suricata/rules
rule-files:
  - suricata.rules
  - /etc/suricata/rules/local.rules
```

### 3. Set Interface to Loopback
In the `af-packet` section of `suricata.yaml`:

```yaml
af-packet:
  - interface: lo
    buffer-size: 32768
    block-size: 32768
    ring-size: 2048
```

Also lower the loopback MTU to avoid AF_PACKET socket errors:

```bash
sudo ip link set lo mtu 1500
```

### 4. Test Configuration
```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

Expected output:
```
Notice - This is Suricata version 6.0.10 RELEASE running in SYSTEM mode
Notice - Configuration provided was successfully loaded. Exiting.
```

---

## Phase 1 — IDS Mode

In IDS mode, Suricata **detects and alerts** but does not block traffic.

### Write a Custom Detection Rule

```bash
sudo nano /etc/suricata/rules/local.rules
```

```
alert icmp any any -> any any (msg:"ICMP Ping Detected"; itype:8; sid:1000001; rev:1;)
```

### Start Suricata in IDS Mode

```bash
sudo suricata -c /etc/suricata/suricata.yaml -i lo
```

### Monitor Alerts

```bash
sudo tail -f /var/log/suricata/fast.log
```

### Trigger the Alert

```bash
ping -c 3 127.0.0.1
```

### Result

```
05/26/2026-13:33:05.403912  [**] [1:1000001:1] ICMP Ping Detected [**] [Classification: (null)] [Priority: 3] {ICMP} 127.0.0.1:8 -> 127.0.0.1:0
05/26/2026-13:33:06.415400  [**] [1:1000001:1] ICMP Ping Detected [**] [Classification: (null)] [Priority: 3] {ICMP} 127.0.0.1:8 -> 127.0.0.1:0
05/26/2026-13:33:07.428671  [**] [1:1000001:1] ICMP Ping Detected [**] [Classification: (null)] [Priority: 3] {ICMP} 127.0.0.1:8 -> 127.0.0.1:0
```

✅ **IDS successfully detected ICMP ping traffic and generated alerts.**

> Alerts appear twice per ping because the loopback interface captures both outgoing and incoming packets.

---

## Phase 2 — IPS Mode

In IPS mode, Suricata **actively blocks** traffic using NFQ (Netfilter Queue).

### 1. Redirect Traffic to Suricata via iptables

```bash
sudo iptables -I INPUT -p icmp -j NFQUEUE --queue-num 0
sudo iptables -I OUTPUT -p icmp -j NFQUEUE --queue-num 0
```

### 2. Update Rule to DROP

```bash
sudo nano /etc/suricata/rules/local.rules
```

Change `alert` to `drop`:

```
drop icmp any any -> any any (msg:"ICMP Ping Blocked"; itype:8; sid:1000001; rev:2;)
```

### 3. Start Suricata in IPS/NFQ Mode

```bash
sudo suricata -c /etc/suricata/suricata.yaml -q 0
```

### 4. Test Blocking

```bash
ping -c 3 127.0.0.1
```

### Result

```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
--- 127.0.0.1 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2029ms
```

✅ **IPS successfully blocked all ICMP ping packets — 100% packet loss confirmed.**

---

## Cleanup

Remove iptables rules to restore normal traffic:

```bash
sudo iptables -D INPUT -p icmp -j NFQUEUE --queue-num 0
sudo iptables -D OUTPUT -p icmp -j NFQUEUE --queue-num 0
```

Verify ping works again:

```bash
ping -c 3 127.0.0.1
```

---

## Summary

| Phase | Mode | Rule Action | Ping Result |
|-------|------|-------------|-------------|
| 1 | IDS (passive) | `alert` | Detected, allowed |
| 2 | IPS (inline) | `drop` | Blocked, 100% loss |

### Key Concepts Demonstrated
- **IDS vs IPS** — detection-only vs active blocking
- **Custom Suricata rules** — writing rules with `alert` and `drop` actions
- **NFQ mode** — using Netfilter Queue to intercept packets inline
- **Rule anatomy** — `action proto src dst (options; sid; rev;)`

---

*Lab performed on Parrot OS lory — Suricata 6.0.10*
