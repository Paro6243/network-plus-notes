# Lab 01 — Wireshark Packet Analysis: OSI Model in Action

**Date:** May 2026
**Tool:** Wireshark
**OS:** Parrot OS
**Difficulty:** Beginner
**Topic:** OSI Model, Network Protocols, Encryption, Traffic Analysis

---

## 🎯 Objective

Capture and analyze real network traffic from two different websites (Google and GitHub) using Wireshark, and observe how the OSI model layers appear in actual packets.

---

## 🛠️ Setup

- **OS:** Parrot OS
- **Tool:** Wireshark (built-in on Parrot)
- **Method:** Live packet capture on active network interface
- **Targets:** `google.com` and `github.com`

---

## 🔬 What I Did

1. Opened Wireshark and started a live capture on the active network interface
2. Browsed to `google.com` in the browser while capture was running
3. Stopped capture and saved the `.pcapng` file
4. Repeated the same process for `github.com`
5. Compared the two captures side by side

---

## 👀 Initial Observations

| Property | Google | GitHub |
|----------|--------|--------|
| Protocol visible | QUIC / HTTP3 | TLS 1.3 over TCP |
| IP version | IPv6 | IPv4 |
| Encryption visible? | Looked like plain text at first | Clearly encrypted (TLS handshake visible) |
| Transport layer | UDP (via QUIC) | TCP |

---

## 💡 My Initial Hypothesis (Wrong — and That's Okay)

At first glance, GitHub's traffic looked *more* secure because I could clearly see the TLS handshake and encryption labels in Wireshark. Google's traffic looked almost "open" or plain in comparison.

**My initial conclusion:** GitHub is connecting more securely than Google.

---

## ✅ What I Actually Learned

This was a great lesson in **not judging security by appearance in Wireshark.**

**Google was actually MORE secure and more advanced:**

- Google was using **QUIC** — a modern protocol built on **UDP** developed by Google itself
- QUIC runs over **HTTP/3** and has **encryption built into the transport layer** — unlike TLS which sits on top of TCP
- Because QUIC encrypts everything including the handshake metadata, it *looks* different in Wireshark — less "obvious" TLS labels
- Google was also using **IPv6**, which is the modern standard and supports better routing and security features
- GitHub was using standard **TLS 1.3 over TCP** — still very secure, but older architecture than QUIC

**The lesson:** More visible encryption labels in Wireshark doesn't mean more secure. QUIC/HTTP3 hides more by design.

---

## 🧱 OSI Model Mapping (What I Saw in Wireshark)

| OSI Layer | Google (QUIC) | GitHub (TLS/TCP) |
|-----------|--------------|-----------------|
| Layer 7 — Application | HTTP/3 | HTTPS |
| Layer 6 — Presentation | Encrypted (QUIC) | TLS 1.3 |
| Layer 5 — Session | Managed by QUIC | TCP Session |
| Layer 4 — Transport | **UDP** | **TCP** |
| Layer 3 — Network | **IPv6** | IPv4 |
| Layer 2 — Data Link | Ethernet frame | Ethernet frame |
| Layer 1 — Physical | Network interface | Network interface |

---

## 🔑 Key Takeaways

1. **QUIC is the future** — HTTP/3 + QUIC is faster and more secure than HTTP/2 + TLS/TCP
2. **Wireshark shows you reality** — what looks "open" may actually be more encrypted
3. **IPv6 vs IPv4** — Google's use of IPv6 shows enterprise-grade modern infrastructure
4. **OSI model is real** — every layer showed up in the actual capture, not just theory
5. **Don't assume** — always analyze before concluding in security work

---

## 📁 Captured Files

- `google-capture.pcapng` — Google traffic showing QUIC/HTTP3/IPv6
- `github-capture.pcapng` — GitHub traffic showing TLS 1.3/TCP/IPv4

---

## 🔗 Related Topics

- CompTIA N+ Domain 1: Networking Concepts
- OSI Model (Professor Messer Section 1.1)
- QUIC Protocol — RFC 9000
- TLS 1.3 — RFC 8446

---

*Lab documented as part of Network+ study journey | github.com/Paro6243*
