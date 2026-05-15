# Lab 02 — DNS Configuration & Analysis

**Date:** May 15, 2026
**Tool:** nslookup, Terminal
**OS:** Parrot OS
**Difficulty:** Beginner
**Topic:** DNS Records, Name Resolution, Reverse Lookup
**Reference:** Professor Messer N+ — DNS Configuration

---

## 🎯 Objective

Perform real DNS queries using `nslookup` to understand how DNS record types work in practice — A, AAAA, MX, NS, and PTR records — and observe how Google's DNS infrastructure is built.

---

## 🔬 Lab Commands & Real Output

### 1. Basic A Record Lookup
```bash
$ nslookup google.com

Server:         10.155.101.43
Address:        10.155.101.43#53

Non-authoritative answer:
Name: google.com  Address: 142.250.134.101
Name: google.com  Address: 142.250.134.139
Name: google.com  Address: 142.250.134.113
Name: google.com  Address: 142.250.134.138
Name: google.com  Address: 142.250.134.100
Name: google.com  Address: 142.250.134.102
Name: google.com  Address: 2404:6800:4000:100c::64
Name: google.com  Address: 2404:6800:4000:100c::8b
Name: google.com  Address: 2404:6800:4000:100c::8a
Name: google.com  Address: 2404:6800:4000:100c::71
```

**What I observed:**
- Google returned **6 different IPv4 addresses** — this is load balancing
- Google also returned **IPv6 (AAAA) addresses** automatically
- Response came from local DNS server `10.155.101.43` (my ISP/router)
- One timeout error occurred but resolved — normal behavior

---

### 2. MX Records (Mail Servers)
```bash
$ nslookup -type=MX google.com

Server:         10.155.101.43
Address:        10.155.101.43#53

Non-authoritative answer:
google.com    mail exchanger = 10 smtp.google.com.

Authoritative answers:
smtp.google.com  internet address = 192.178.211.26
smtp.google.com  internet address = 192.178.211.27
smtp.google.com  has AAAA address 2404:6800:4000:1025::1b
smtp.google.com  has AAAA address 2404:6800:4000:1025::1a
```

**What I observed:**
- All email to `@google.com` routes through `smtp.google.com`
- Priority value `10` — lower number = higher priority
- Mail server also has both IPv4 and IPv6 addresses

---

### 3. Querying via Google's Public DNS (8.8.8.8)
```bash
$ nslookup google.com 8.8.8.8

Server:         8.8.8.8
Address:        8.8.8.8#53

Non-authoritative answer:
Name: google.com  Address: 142.250.134.102
Name: google.com  Address: 142.250.134.100
Name: google.com  Address: 142.250.134.139
Name: google.com  Address: 142.250.134.101
Name: google.com  Address: 142.250.134.138
Name: google.com  Address: 142.250.134.113
Name: google.com  Address: 2404:6800:4009:803::200e
```

**What I observed:**
- Bypassed my ISP DNS and queried Google's own DNS directly
- Same IPs returned — confirms DNS consistency globally
- Server line now shows `8.8.8.8` instead of my local DNS

---

### 4. NS Records (Nameservers)
```bash
$ nslookup -type=NS google.com

Server:         10.155.101.43
Address:        10.155.101.43#53

Non-authoritative answer:
google.com  nameserver = ns4.google.com.
google.com  nameserver = ns2.google.com.
google.com  nameserver = ns1.google.com.
google.com  nameserver = ns3.google.com.

Authoritative answers:
ns4.google.com  internet address = 216.239.38.10
ns4.google.com  has AAAA address 2001:4860:4802:38::a
ns2.google.com  internet address = 216.239.34.10
ns2.google.com  has AAAA address 2001:4860:4802:34::a
ns1.google.com  internet address = 216.239.32.10
ns1.google.com  has AAAA address 2001:4860:4802:32::a
ns3.google.com  internet address = 216.239.36.10
ns3.google.com  has AAAA address 2001:4860:4802:36::a
```

**What I observed:**
- Google runs **4 nameservers** (ns1–ns4) for redundancy
- If one goes down, the other 3 handle DNS resolution
- All nameservers have both IPv4 and IPv6 addresses

---

### 5. Reverse DNS Lookup (PTR Record)
```bash
$ nslookup 8.8.8.8

8.8.8.8.in-addr.arpa    name = dns.google.
```

**What I observed:**
- Gave an IP, got back a hostname — this is a PTR record
- `8.8.8.8` resolves to `dns.google` — confirms it's Google's DNS server
- Used in network troubleshooting to identify unknown IPs

---

## 📊 DNS Records Summary

| Record | Command Used | Result |
|--------|-------------|--------|
| A | `nslookup google.com` | 6 IPv4 addresses (load balanced) |
| AAAA | `nslookup google.com` | IPv6 addresses returned automatically |
| MX | `nslookup -type=MX google.com` | smtp.google.com (priority 10) |
| NS | `nslookup -type=NS google.com` | ns1–ns4.google.com |
| PTR | `nslookup 8.8.8.8` | dns.google |

---

## 💡 Key Takeaways

1. **Load balancing is real** — Google returns multiple IPs to distribute traffic globally
2. **MX records control email routing** — not the A record
3. **4 nameservers = redundancy** — enterprise DNS never has a single point of failure
4. **Reverse DNS (PTR)** is a powerful troubleshooting tool — IP → hostname
5. **Non-authoritative answer** means the response came from cache, not the original nameserver
6. **DNS runs on UDP port 53** — fast and connectionless by design
7. **Querying different DNS servers** (ISP vs 8.8.8.8) returns consistent results — DNS is globally synchronized

---

## 🔗 Related Topics

- CompTIA N+ Domain 1: Networking Concepts
- DNS over HTTPS (DoH) — RFC 8484
- DNSSEC — DNS Security Extensions
- Professor Messer N+ Section: DNS Configuration

---

*Lab performed on Parrot OS | Part of Network+ study journey | [github.com/Paro6243](https://github.com/Paro6243)*
