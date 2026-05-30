# Suricata IDS/IPS Lab on Parrot OS

## Overview
This lab demonstrates setting up Suricata as both an **Intrusion Detection System (IDS)** and an **Intrusion Prevention System (IPS)** on a single machine using the loopback interface.

| Detail | Value |
|--------|-------|
| OS | Parrot OS (lory) |
| Tool | Suricata 6.0.10 |
| Interface | Loopback (`lo`) |
| Phases | IDS → IPS → Nmap Detection → HTTP Detection |

---

## Table of Contents
- [Installation](#installation)
- [Configuration](#configuration)
- [Phase 1 — IDS Mode](#phase-1--ids-mode)
- [Phase 2 — IPS Mode](#phase-2--ips-mode)
- [Phase 3 — Nmap Port Scan Detection](#phase-3--nmap-port-scan-detection)
- [Phase 4 — HTTP Attack Detection](#phase-4--http-attack-detection)
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
    checksum-checks: no
```

Also lower the loopback MTU to avoid AF_PACKET socket errors:

```bash
sudo ip link set lo mtu 1500
```

> **Note:** MTU resets after every reboot. Always run this before starting Suricata on loopback.

### 4. Fix HTTP App Layer Detection Port
By default Suricata only inspects HTTP on port 80. Add port 8080:

```bash
sudo python3 -c "
content = open('/etc/suricata/suricata.yaml').read()
content = content.replace('    http:\n      enabled: yes', '    http:\n      enabled: yes\n      detection-ports:\n        dp: 80, 8080')
open('/etc/suricata/suricata.yaml', 'w').write(content)
"
```

### 5. Disable Checksum Validation
Loopback uses checksum offloading which causes Suricata to skip app layer inspection:

```bash
sudo sed -i '647s/checksum-checks:.*/checksum-checks: no/' /etc/suricata/suricata.yaml
```

### 6. Test Configuration
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

**Rule breakdown:**
- `alert` — action: log an alert
- `icmp` — protocol to match
- `any any -> any any` — match any source to any destination
- `itype:8` — ICMP type 8 = echo request (ping)
- `sid:1000001` — unique rule ID (custom rules use 1000000+)

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

> `iptables` intercepts ICMP packets and sends them to queue 0. Suricata listens on that queue and decides to allow or drop each packet.

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

## Phase 3 — Nmap Port Scan Detection

Port scans are one of the first steps in any attack — the attacker maps what services are running. Suricata can detect different scan types based on TCP flag patterns.

### How Scan Types Work

| Scan Type | TCP Flags | Why Attackers Use It |
|-----------|-----------|---------------------|
| SYN scan | SYN only (no ACK) | Fast, stealthy, doesn't complete handshake |
| NULL scan | No flags at all | Bypasses some stateless firewalls |
| FIN scan | FIN only | Bypasses some stateless firewalls |

### Rules

```bash
sudo nano /etc/suricata/rules/local.rules
```

```
alert tcp any any -> any any (msg:"Nmap SYN Scan Detected"; flags:S,12; threshold: type threshold, track by_src, count 5, seconds 2; sid:1000002; rev:2;)
alert tcp any any -> any any (msg:"Nmap NULL Scan Detected"; flags:0; threshold: type threshold, track by_src, count 5, seconds 2; sid:1000003; rev:2;)
alert tcp any any -> any any (msg:"Nmap FIN Scan Detected"; flags:F,12; threshold: type threshold, track by_src, count 5, seconds 2; sid:1000004; rev:2;)
```

**Key options:**
- `flags:S,12` — match SYN flag, checking only specific bits (more reliable on loopback)
- `flags:0` — match packets with zero flags (NULL scan)
- `threshold: count 5, seconds 2` — only alert after 5 packets in 2 seconds (reduces false positives)

### Trigger the Scans

```bash
# SYN scan — most common
sudo nmap -sS 127.0.0.1

# NULL scan — no flags
sudo nmap -sN 127.0.0.1

# FIN scan — stealth technique
sudo nmap -sF 127.0.0.1
```

### Result

```
05/28/2026-14:03:28.432494  [**] [1:1000003:1] Nmap NULL Scan Detected [**] ... 127.0.0.1:51113 -> 127.0.0.1:139
05/28/2026-14:03:28.432564  [**] [1:1000003:1] Nmap NULL Scan Detected [**] ... 127.0.0.1:51113 -> 127.0.0.1:443
05/28/2026-14:36:04.058800  [**] [1:1000004:1] Nmap FIN Scan Detected [**]  ... 127.0.0.1:43753 -> 127.0.0.1:45100
```

### Alert Counts

| Scan Type | Alerts |
|-----------|--------|
| SYN Scan | 99 (threshold limited) |
| NULL Scan | 1985 |
| FIN Scan | 2002 |

✅ **All 3 scan types successfully detected.**

> SYN count is lower because of the threshold — this is intentional to reduce noise in production.

---

## Phase 4 — HTTP Attack Detection

### Setup — Start HTTP Server

```bash
cd /tmp && python3 -m http.server 8080 &
```

### Rules

```bash
sudo nano /etc/suricata/rules/local.rules
```

```
alert http any any -> any any (msg:"HTTP Directory Traversal Detected"; flow:established,to_server; content:"../"; http_raw_uri; sid:1000005; rev:5;)
alert http any any -> any any (msg:"HTTP SQL Injection Detected"; flow:established,to_server; content:"OR+1%3D1"; http_raw_uri; nocase; sid:1000006; rev:3;)
alert http any any -> any any (msg:"Suspicious User-Agent sqlmap"; flow:established,to_server; http.user_agent; content:"sqlmap"; nocase; sid:1000007; rev:2;)
alert http any any -> any any (msg:"Suspicious User-Agent nikto"; flow:established,to_server; http.user_agent; content:"nikto"; nocase; sid:1000008; rev:2;)
```

**Key options:**
- `flow:established,to_server` — only match established connections going to server (required for HTTP rules on loopback)
- `http_raw_uri` — inspect raw unprocessed URI before normalization
- `http.user_agent` — inspect the HTTP User-Agent header
- `nocase` — case insensitive matching

### Trigger the Attacks

```bash
# Directory traversal
curl --path-as-is http://127.0.0.1:8080/../../../etc/passwd

# SQL injection
curl "http://127.0.0.1:8080/?id=1+OR+1%3D1"

# sqlmap scanner simulation
curl -A "sqlmap/1.0" http://127.0.0.1:8080/

# nikto scanner simulation
curl -A "nikto/2.1.6" http://127.0.0.1:8080/
```

### Result

```
05/29/2026-13:34:46.901657  [**] [1:1000007:2] Suspicious User-Agent sqlmap [**] ... 127.0.0.1:38366 -> 127.0.0.1:8080
05/29/2026-16:17:04.430172  [**] [1:1000007:2] Suspicious User-Agent sqlmap [**] ... 127.0.0.1:49552 -> 127.0.0.1:8080
05/29/2026-16:17:23.764335  [**] [1:1000008:2] Suspicious User-Agent nikto [**]  ... 127.0.0.1:49890 -> 127.0.0.1:8080
```

✅ **HTTP scanners and attack patterns successfully detected.**

> **Troubleshooting note:** HTTP rules require `flow:established,to_server` to work on loopback. Without it, the rules load but never fire. Also ensure checksum validation is disabled and port 8080 is added to HTTP detection ports.

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

| Phase | Attack Type | Detection Method | Result |
|-------|-------------|-----------------|--------|
| 1 | ICMP Ping | `itype:8` rule | ✅ Detected |
| 2 | ICMP Ping | `drop` + NFQ | ✅ Blocked |
| 3 | Nmap SYN/NULL/FIN scan | TCP flags matching | ✅ Detected |
| 4 | HTTP traversal, SQLi, scanners | HTTP app layer rules | ✅ Detected |

### Key Concepts Demonstrated
- **IDS vs IPS** — detection-only vs active blocking
- **Custom Suricata rules** — writing rules with `alert` and `drop` actions
- **NFQ mode** — using Netfilter Queue to intercept packets inline
- **TCP flag analysis** — detecting port scans by flag patterns
- **HTTP app layer inspection** — detecting attacks inside HTTP traffic
- **Rule anatomy** — `action proto src dst (options; sid; rev;)`
- **Loopback quirks** — MTU issues, checksum offloading, flow direction requirements

---

*Lab performed on Parrot OS lory — Suricata 6.0.10*
