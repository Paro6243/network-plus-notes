# Lab 02 — DNS Configuration & Analysis

**Date:** May 2026
**Tool:** Terminal, nslookup, Wireshark
**OS:** Parrot OS
**Difficulty:** Beginner
**Topic:** DNS, Name Resolution, DNS Record Types
**Reference:** Professor Messer N+ — DNS Configuration

---

## 🎯 Objective

Understand how DNS works in practice — how domain names get resolved to IP addresses, what DNS record types exist, and how to query and analyze DNS traffic using command line tools.

---

## 📖 Key Concepts (From Prof Messer)

### What is DNS?
DNS (Domain Name System) is the internet's "phone book." It translates human-readable domain names like `google.com` into IP addresses like `142.250.195.46` that computers use to communicate.

### DNS Resolution Process (Step by Step)

```
Your Browser
     │
     ▼
1. Check local cache — already know the IP?
     │ No
     ▼
2. Ask Recursive Resolver (your ISP or 8.8.8.8)
     │
     ▼
3. Resolver asks Root DNS Server → "Who handles .com?"
     │
     ▼
4. Root says → "Ask the .com TLD server"
     │
     ▼
5. TLD server says → "Ask google.com's authoritative server"
     │
     ▼
6. Authoritative server → returns the IP address ✅
     │
     ▼
7. Resolver caches it and sends IP back to your browser
```

### DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Maps domain → IPv4 address | `google.com → 142.250.195.46` |
| **AAAA** | Maps domain → IPv6 address | `google.com → 2607:f8b0::...` |
| **CNAME** | Alias — points to another domain | `www.google.com → google.com` |
| **MX** | Mail server for a domain | `google.com → smtp.google.com` |
| **TXT** | Text info (used for verification, SPF) | `"v=spf1 include:..."` |
| **NS** | Nameserver for a domain | `google.com → ns1.google.com` |
| **PTR** | Reverse lookup — IP → domain | `142.250.195.46 → google.com` |
| **SOA** | Start of Authority — zone info | Admin contact, serial number |

---

## 🔬 Lab — DNS Queries with nslookup

### Basic A Record Lookup
```bash
$ nslookup google.com

Server:         192.168.1.1
Address:        192.168.1.1#53

Non-authoritative answer:
Name:   google.com
Address: 142.250.195.46        ← IPv4 (A record)
Name:   google.com
Address: 2607:f8b0::200e       ← IPv6 (AAAA record)
```

### Query a Specific Record Type
```bash
# MX records (mail servers)
$ nslookup -type=MX google.com

# NS records (nameservers)
$ nslookup -type=NS google.com

# TXT records
$ nslookup -type=TXT google.com
```

### Query a Specific DNS Server
```bash
# Ask Google's public DNS directly
$ nslookup google.com 8.8.8.8

# Ask Cloudflare's DNS
$ nslookup google.com 1.1.1.1
```

### Reverse DNS Lookup (PTR record)
```bash
$ nslookup 8.8.8.8

Non-authoritative answer:
8.8.8.8.in-addr.arpa  name = dns.google
```

---

## 🔍 DNS in Wireshark

DNS traffic runs on **UDP Port 53** (sometimes TCP for large responses).

**Filter to see only DNS traffic:**
```
dns
```

**What you see in a DNS packet:**
- **Query:** Your machine asking "What is the IP for google.com?"
- **Response:** DNS server replying with the IP
- **Transaction ID:** Matches query to response
- **TTL:** How long to cache the answer (in seconds)

---

## 💡 Key Takeaways

1. **DNS uses UDP port 53** by default — fast, connectionless
2. **TTL matters** — low TTL = frequent lookups, high TTL = cached longer
3. **DNS is unencrypted by default** — DNS over HTTPS (DoH) fixes this
4. **nslookup is your best friend** for quick DNS troubleshooting
5. **Non-authoritative answer** = response came from cache, not the original nameserver
6. **DNS poisoning** is a real attack — knowing DNS deeply is key for security

---

## 🔗 Related Topics

- CompTIA N+ Domain 1: Networking Concepts
- DNS over HTTPS (DoH) — RFC 8484
- DNSSEC — DNS Security Extensions
- Professor Messer N+ Section: DNS Configuration

---

*Lab documented as part of Network+ study journey | [github.com/Paro6243](https://github.com/Paro6243)*
