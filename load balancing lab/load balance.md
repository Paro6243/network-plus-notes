# Lab 05 — Load Balancing with Nginx

**Platform:** Parrot OS  
**Tools:** Nginx, Python HTTP Server, tcpdump, Wireshark  
**Network+ Objectives:** 2.2 (Network Implementations), 1.1 (TCP/IP Protocols)

---

## Objective

Simulate a software load balancer distributing HTTP traffic across multiple backend servers. Observe load balancing behaviour at the network level using packet capture, and test automatic failover when a backend goes down.

---

## Lab Topology

```
Client (curl) ──► Nginx :8080 ──► Backend Server 1 :8001
                               ──► Backend Server 2 :8002
                               ──► Backend Server 3 :8003
```

---

## Concepts Covered

### What is a Load Balancer?

A load balancer sits between clients and backend servers, distributing incoming requests so no single server is overwhelmed. It provides:

- **Scalability** — traffic spread across multiple servers
- **High Availability** — if one server dies, traffic automatically reroutes
- **Performance** — prevents any single server from becoming a bottleneck

### Key Network Insight — Two TCP Connections Per Request

```
Client ──[TCP SYN]──► Nginx :8080       (Connection 1)
Nginx  ──[TCP SYN]──► Backend :8001     (Connection 2)
```

The client never talks directly to the backend. Nginx terminates the client connection and creates a separate connection to the chosen backend. This was confirmed via Wireshark packet capture.

---

## Load Balancing Algorithms

### Round Robin (Default)
Distributes requests in a fixed rotation — Server 1, Server 2, Server 3, Server 1...

**Best for:** Short-lived requests, stateless HTTP APIs, simple web servers.

### Least Connections
Sends the next request to whichever backend currently has the fewest active connections.

**Best for:** Long-lived connections — databases, file downloads, video streaming — where connection hold time varies significantly between requests.

> **Note:** Under fast/short-lived requests (like curl), least connections behaves identically to round robin because all servers drop to 0 active connections between requests. The difference only appears under sustained load with slow backends.

---

## Setup

### Step 1 — Install Nginx

```bash
sudo apt update && sudo apt install nginx -y
sudo systemctl start nginx
sudo nginx -v
```

> HAProxy was the original tool but failed due to missing `libopentracing-c-wrapper0` dependency not packaged in Parrot OS repos. Nginx upstream module achieves identical results.

### Step 2 — Start 3 Backend Servers

Simulated using Python's built-in HTTP server:

```bash
# Terminal 1
mkdir -p ~/lb-lab/server1 && echo "Response from Server 1" > ~/lb-lab/server1/index.html
cd ~/lb-lab/server1 && python3 -m http.server 8001

# Terminal 2
mkdir -p ~/lb-lab/server2 && echo "Response from Server 2" > ~/lb-lab/server2/index.html
cd ~/lb-lab/server2 && python3 -m http.server 8002

# Terminal 3
mkdir -p ~/lb-lab/server3 && echo "Response from Server 3" > ~/lb-lab/server3/index.html
cd ~/lb-lab/server3 && python3 -m http.server 8003
```

Verify all backends respond:

```bash
curl http://localhost:8001/index.html
curl http://localhost:8002/index.html
curl http://localhost:8003/index.html
```

### Step 3 — Configure Nginx as Load Balancer

Edit `/etc/nginx/nginx.conf` and add inside the `http {}` block:

```nginx
upstream backend_servers {
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}

server {
    listen 8080;
    location / {
        proxy_pass http://backend_servers;
    }
}
```

Test config and restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

---

## Tests & Results

### Test 1 — Round Robin Distribution

```bash
for i in {1..9}; do curl -s http://localhost:8080/index.html; done
```

**Result:**
```
Response from Server 1
Response from Server 1
Response from Server 1
Response from Server 1
Response from Server 2
Response from Server 3
Response from Server 1
Response from Server 2
Response from Server 3
```

First 4 requests hit Server 1 due to Nginx keepalive reusing the existing TCP connection. After keepalive timeout, rotation began across all 3 servers.

---

### Test 2 — Failover (Server 2 killed)

Killed Server 2 (`Ctrl+C`), then re-ran curl loop:

```bash
for i in {1..9}; do curl -s http://localhost:8080/index.html; done
```

**Result:**
```
Response from Server 1
Response from Server 3
Response from Server 3
Response from Server 1
Response from Server 3
Response from Server 1
Response from Server 3
Response from Server 1
Response from Server 3
```

Server 2 completely absent. Nginx detected the dead backend and automatically rerouted all traffic to Server 1 and Server 3 — no manual intervention required.

---

### Test 3 — Packet Capture (tcpdump + Wireshark)

```bash
sudo tcpdump -i lo port 8080 or port 8001 or port 8002 or port 8003 -w ~/lb-lab/lb_capture.pcapng
```

Ran curl loop during capture, then analysed in Wireshark.

**Observation:** Wireshark confirmed two separate TCP connections per request:
- Client → Nginx on port 8080
- Nginx → Backend on port 8001 or 8003

This visually proved that the load balancer acts as an intermediary, never exposing backend servers directly to clients.

### Wireshark Capture Analysis

![Wireshark capture](wireshark-capture.png)

**Packet-by-packet breakdown:**

| Packets | Flow | What it shows |
|---|---|---|
| 1-2 | `::1 → ::1` port 8080 | IPv6 attempt → RST (Nginx not listening on IPv6) |
| 3-5 | `127.0.0.1 → 8080` | TCP 3-way handshake — Client connects to Nginx |
| 6 | `34678 → 8080` | HTTP GET request from curl to Nginx |
| 8-10 | `127.0.0.1 → 8001` | Second TCP handshake — Nginx connects to Backend 1 |
| 11 | `54732 → 8001` | Nginx forwards GET request to backend |
| 13-14 | `8001 → 54732` | Backend sends response back to Nginx |

Two complete TCP handshakes confirm the load balancer acts as a **full proxy** — clients and backends never communicate directly.

---

### Test 4 — Least Connections Algorithm

Changed algorithm in nginx.conf:

```nginx
upstream backend_servers {
    least_conn;
    server 127.0.0.1:8001;
    server 127.0.0.1:8002;
    server 127.0.0.1:8003;
}
```

**Result:** Identical distribution to round robin under fast curl requests (expected — see Concepts section above).

---

## Troubleshooting Notes

| Issue | Cause | Fix |
|---|---|---|
| HAProxy failed to install | Missing `libopentracing-c-wrapper0` not in Parrot repos | Switched to Nginx upstream module |
| `nginx` command not found | Binary at `/usr/sbin/nginx`, requires sudo | Use `sudo nginx -v` |
| Nginx inactive after install | Not auto-started | `sudo systemctl start nginx` |
| First 4 requests hit same server | Nginx keepalive reusing TCP connection | Expected behaviour, not a bug |

---

## Key Takeaways

- A load balancer creates **two separate TCP connections** per request — client never reaches backend directly
- **Round robin** is suitable for short stateless requests; **least connections** for long-lived or variable-duration connections
- Nginx detects dead backends automatically — no config change needed for failover
- Software load balancers (Nginx, HAProxy) replicate what hardware load balancers (F5, Citrix) do in enterprise environments
- Dependency resolution is a real sysadmin skill — when HAProxy failed, identifying and switching to Nginx was the correct professional response

---

## Files

```
lb-lab/
├── server1/index.html
├── server2/index.html
├── server3/index.html
└── lb_capture.pcapng
```

---

*Part of Network+ home lab series — Parrot OS*
