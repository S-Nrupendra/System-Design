# System Design — Introduction

## What is System Design?

System design is the process of defining the architecture, components, data flow, and interactions of a system to satisfy given requirements. In interviews, you're asked to design large-scale systems like "Design WhatsApp" or "Design YouTube" — and you need to reason through trade-offs, not just list components.

It sits at the intersection of:
- **Software architecture** — how components are structured
- **Distributed systems** — how they communicate and stay consistent
- **Engineering trade-offs** — why you pick X over Y

---

## Two Types of Requirements You'll Always Face

| | Functional | Non-functional |
|---|---|---|
| What it means | What the system *does* | How well it does it |
| Examples | "User can send a message" | "System handles 1M users, 99.9% uptime" |
| Your job | Clarify and scope | Estimate and design for |

Non-functional requirements (NFRs) are what separate a senior answer from a junior one.

---

## The Standard Interview Framework

(Use this every single time)

1. **Clarify requirements** — ask questions, scope it down
2. **Estimate scale** — users, requests/sec, storage, bandwidth
3. **Define the API** — what endpoints/interfaces does it expose?
4. **High-level design** — draw the major components
5. **Deep dive** — pick the hard parts (DB schema, caching strategy, etc.)
6. **Identify bottlenecks** — where does it break at scale? How do you fix it?

---

## The Mental Shift You Need

In coding rounds, there's a correct answer. In system design, there's no single right answer — only **justified trade-offs**. An interviewer wants to hear *why* you made a choice, not just *what* you chose.

> "I'm using a relational DB here because writes are transactional and data is highly structured — though if reads scaled beyond X, I'd consider adding a read replica or moving to a NoSQL store."

That kind of reasoning is what gets you hired.

# Phase 1 — Topic 1: Client-Server Model

---

## What is the Client-Server Model?

The client-server model is a distributed application structure that partitions tasks between **providers of a resource or service (servers)** and **requesters of that resource (clients)**.

- The **client** initiates communication — it sends a request.
- The **server** listens for requests, processes them, and sends back a response.
- Communication happens over a **network** using agreed-upon protocols (HTTP, TCP, etc.)

This is the foundational model behind almost every system you will ever design.

---

## How a Request-Response Cycle Works

```
Client                          Server
  |                               |
  |------- Request (HTTP) ------->|
  |                               |  (process: auth, DB query, business logic)
  |<------ Response (JSON) ------>|
  |                               |
```

### Step-by-step breakdown

1. User types `https://google.com` in a browser (client).
2. Browser performs a **DNS lookup** to resolve the domain to an IP address.
3. A **TCP connection** is established (3-way handshake).
4. Browser sends an **HTTP GET request** to the server.
5. Server processes the request — queries DB, runs logic, builds response.
6. Server sends back an **HTTP response** (status code + body).
7. Browser renders the HTML/CSS/JS.

---

## Types of Clients

| Type | Examples | Characteristics |
|---|---|---|
| Thin client | Web browser, terminal UI | Minimal processing; relies heavily on server |
| Thick/Fat client | Desktop apps, mobile apps | Heavy local processing; server just stores/syncs data |
| Hybrid | Modern SPAs (React apps) | Client handles UI logic; server handles data/auth |

---

## Types of Servers

| Type | Purpose | Examples |
|---|---|---|
| Web server | Serve static files, handle HTTP | Nginx, Apache |
| Application server | Run business logic | Node.js, Django, Spring Boot |
| Database server | Store and retrieve data | PostgreSQL, MySQL, MongoDB |
| Cache server | Store frequently accessed data in memory | Redis, Memcached |
| File/Object storage server | Store blobs, images, videos | AWS S3, GCS |
| Message queue server | Async communication between services | Kafka, RabbitMQ |

In real systems, a single "server" is usually a **cluster of machines** behind a load balancer.

---

## Stateless vs Stateful Servers

This is a critical distinction in system design.

### Stateless server
- Does **not** store any client session data between requests.
- Every request contains all the information needed to process it.
- **Scalable** — any server instance can handle any request.
- Example: REST APIs using JWT tokens (token carries user identity).

### Stateful server
- Remembers client state between requests (stores session in memory).
- A client must always talk to the **same server instance**.
- **Harder to scale** — adding a new instance doesn't help if sessions are tied to one machine.
- Example: Old-school PHP sessions stored in server memory.

> **Rule of thumb:** Always design servers to be stateless. Move state to a shared external store (Redis, DB) instead.

---

## Peer-to-Peer (P2P) vs Client-Server

| | Client-Server | Peer-to-Peer |
|---|---|---|
| Structure | Centralized server | Decentralized nodes |
| Scalability | Server can become a bottleneck | Scales naturally with more peers |
| Reliability | Single point of failure (if no redundancy) | Resilient — no central failure point |
| Examples | Web apps, APIs | BitTorrent, blockchain |
| Complexity | Simpler to build and manage | Complex coordination |

In most system design interviews, you default to client-server. P2P comes up in specific cases like video calling (WebRTC), file sharing, or blockchain.

---

## Communication Patterns Built on Client-Server

### 1. Request-Response (synchronous)
- Client sends a request and **waits** for a response.
- Simple, widely used.
- Problem: client is blocked while waiting.
- Example: REST API call.

### 2. Long Polling
- Client sends a request. Server **holds the connection open** until it has new data, then responds.
- Client immediately sends another request after receiving a response.
- Used when you need near-real-time updates but can't use WebSockets.
- Example: Early versions of chat apps.

### 3. WebSockets (persistent connection)
- A single TCP connection is kept open **bidirectionally**.
- Server can **push data** to client at any time without client requesting.
- Low latency, efficient for real-time systems.
- Example: WhatsApp, live sports scores, collaborative editing (Google Docs).

### 4. Server-Sent Events (SSE)
- One-way: server pushes updates to client over a persistent HTTP connection.
- Simpler than WebSockets when you only need server → client updates.
- Example: Live notifications, stock tickers.

```
Pattern         | Direction         | Connection     | Use case
----------------|-------------------|----------------|----------------------
Request-Response| Client → Server   | Short-lived    | Normal API calls
Long Polling    | Client → Server   | Held open      | Near-real-time updates
WebSockets      | Bidirectional     | Persistent     | Chat, live collaboration
SSE             | Server → Client   | Persistent     | Notifications, feeds
```

---

## Scaling the Server Side

When traffic grows, one server is not enough. Here's how you scale:

### Vertical Scaling (Scale Up)
- Give the server more CPU, RAM, disk.
- Simple — no code changes needed.
- Has a hard upper limit (can't add infinite RAM to one machine).
- Creates a single point of failure.

### Horizontal Scaling (Scale Out)
- Add more server instances.
- Put a **load balancer** in front to distribute traffic.
- Stateless servers are required for this to work cleanly.
- This is the standard approach for large-scale systems.

```
                    ┌──────────────┐
                    │ Load Balancer│
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
      [Server 1]      [Server 2]      [Server 3]
           │               │               │
           └───────────────┴───────────────┘
                           │
                    [Shared DB / Cache]
```

---

## What Can Go Wrong (Failure Modes)

| Problem | Cause | Solution |
|---|---|---|
| Server overload | Too many requests | Load balancer + horizontal scaling |
| Single point of failure | Only one server | Redundancy, failover |
| High latency | Server far from client | CDN, geo-distributed servers |
| Slow DB queries | No indexing, large scans | Caching, indexing, read replicas |
| Network failure | Packet loss, timeout | Retry logic, circuit breakers |

---

## Key Takeaways for Interviews

1. **Always clarify** whether the system needs real-time communication — that determines if you need WebSockets vs plain HTTP.
2. **Design servers to be stateless** — it makes horizontal scaling trivial.
3. When asked "how do you handle more users?" — the answer starts with a load balancer in front of multiple stateless server instances.
4. The client-server model is the skeleton of every system you design — load balancers, CDNs, caches, queues are all just specialized servers.

---

## Next Topic → IP, TCP, UDP

# Phase 1 — Topic 2: IP, TCP, and UDP

---

## Why This Matters in System Design

Every time a client talks to a server, data travels over a network. Understanding how that data is addressed (IP), reliably delivered (TCP), or fast-delivered (UDP) helps you make the right protocol choices — especially for real-time systems, file transfers, video streaming, and gaming.

---

## The Network Layers (Simplified)

Networks are organized in layers. Each layer has a specific job and hands off to the layer above/below it.

```
Layer           | Protocol        | Job
----------------|-----------------|--------------------------------------------
Application     | HTTP, DNS, SMTP | What the data means (your app lives here)
Transport       | TCP, UDP        | How data is delivered (reliable vs fast)
Network         | IP              | How data is routed across networks (addressing)
Data Link       | Ethernet, Wi-Fi | How data moves on a single physical link
Physical        | Cables, radio   | Actual bits on the wire
```

As a system designer, you mostly operate at the **Application and Transport layers**, but you need to know what IP is doing underneath.

---

## IP — Internet Protocol

### What it does
IP is responsible for **addressing and routing** packets from a source machine to a destination machine across multiple networks.

- Every machine on a network has an **IP address** — a unique identifier.
- Data is broken into **packets**. Each packet carries the source IP and destination IP.
- Routers along the path use the destination IP to forward each packet toward the destination.

### Key properties of IP
- **Connectionless** — no setup before sending; just fire packets.
- **Best-effort delivery** — packets can be lost, duplicated, or arrive out of order.
- **No guarantee** — IP alone does not ensure reliable delivery.

This is why TCP and UDP exist — they sit on top of IP and handle delivery differently.

### IPv4 vs IPv6

| | IPv4 | IPv6 |
|---|---|---|
| Format | `192.168.1.1` (32-bit) | `2001:0db8::1` (128-bit) |
| Address space | ~4.3 billion addresses | ~340 undecillion addresses |
| Status | Still dominant but exhausted | Growing adoption |
| NAT needed? | Yes (to extend address space) | No |

---

## TCP — Transmission Control Protocol

### What it does
TCP provides **reliable, ordered, error-checked delivery** of a stream of bytes between two endpoints.

It adds the following guarantees on top of IP:
- **Reliability** — lost packets are retransmitted.
- **Ordering** — packets are reassembled in the correct order.
- **Flow control** — sender doesn't overwhelm the receiver.
- **Congestion control** — sender slows down when the network is congested.
- **Error detection** — checksums verify data integrity.

### The 3-Way Handshake (Connection Setup)

Before any data is sent, TCP establishes a connection:

```
Client                        Server
  |                              |
  |-------- SYN ----------------->|   "I want to connect, my seq = X"
  |                              |
  |<------- SYN-ACK -------------|   "OK, my seq = Y, ack your X+1"
  |                              |
  |-------- ACK ----------------->|   "Acknowledged, ack your Y+1"
  |                              |
  |====== Data flows now =========|
```

- **SYN** = Synchronize (initiate connection)
- **ACK** = Acknowledge (confirm receipt)

This handshake adds **latency** — one round trip before data can flow. This is a real cost in high-performance systems.

### Connection Teardown (4-Way)

```
Client                        Server
  |-------- FIN ----------------->|   "I'm done sending"
  |<------- ACK -----------------|
  |<------- FIN -----------------|   "I'm done too"
  |-------- ACK ----------------->|
```

### How TCP Ensures Reliability

1. Every byte sent is assigned a **sequence number**.
2. The receiver sends back an **ACK** (acknowledgement) for bytes received.
3. If the sender doesn't receive an ACK within a timeout, it **retransmits**.
4. The receiver uses sequence numbers to **reorder** out-of-order packets.
5. **Sliding window** allows multiple packets in-flight at once without waiting for each ACK (improves throughput).

### TCP Use Cases
- HTTP/HTTPS (web browsing)
- Email (SMTP, IMAP)
- File transfers (FTP, SSH)
- Databases (MySQL, PostgreSQL connections)
- Anything where data correctness matters more than speed

---

## UDP — User Datagram Protocol

### What it does
UDP is a **connectionless, fire-and-forget** protocol. It sends packets (called datagrams) with no handshake, no acknowledgement, and no retransmission.

### What UDP does NOT provide
- No connection setup
- No guaranteed delivery
- No ordering
- No retransmission of lost packets
- No flow/congestion control

### Why would you ever use UDP then?

Because **speed matters more than perfection** in many applications:

- A video call can tolerate a dropped frame — it just looks slightly blurry for a moment.
- A retransmitted frame from 200ms ago is useless — you'd rather have the current frame, even if imperfect.
- UDP lets the **application layer decide** how to handle loss, rather than imposing TCP's overhead.

### UDP Use Cases
- **Video/audio streaming** — YouTube Live, Zoom, Google Meet
- **Online gaming** — player position updates (stale data is useless)
- **DNS** — small, single request-response lookups; fast matters
- **VoIP** — real-time voice
- **IoT sensors** — high-frequency small packets where some loss is fine
- **QUIC protocol** — HTTP/3 is built on UDP with custom reliability on top

---

## TCP vs UDP — Side-by-Side

| Property | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (handshake) | Connectionless |
| Reliability | Guaranteed delivery | No guarantee |
| Ordering | Ordered | Unordered |
| Speed | Slower (overhead) | Faster (minimal overhead) |
| Error checking | Yes (retransmission) | Checksum only, no retransmit |
| Flow control | Yes | No |
| Header size | 20 bytes | 8 bytes |
| Use when | Correctness matters | Speed/latency matters |
| Examples | HTTP, SSH, DB connections | DNS, video calls, gaming |

---

## Ports

Both TCP and UDP use **port numbers** to identify which application on a machine should receive a packet.

- IP address identifies the **machine**.
- Port number identifies the **application/process** on that machine.

```
Destination: 192.168.1.10:443
              └────────┘ └─┘
              IP address  Port (HTTPS)
```

### Well-known port numbers

| Port | Protocol | Used for |
|---|---|---|
| 22 | TCP | SSH |
| 53 | UDP/TCP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |
| 27017 | TCP | MongoDB |

---

## Sockets

A **socket** is the combination of an IP address + port number. It's the endpoint of a network connection.

```
Socket = IP Address + Port
Example: 192.168.1.10:8080
```

A TCP connection is uniquely identified by a **4-tuple**:
```
(Source IP, Source Port, Destination IP, Destination Port)
```

This is how a server can handle thousands of simultaneous clients — each connection has a unique 4-tuple even though the server's IP and port (e.g., :443) stay the same.

---

## What is QUIC / HTTP/3?

QUIC is a modern transport protocol developed by Google, now standardized. It is the foundation of **HTTP/3**.

- Built **on top of UDP** (not TCP)
- Implements its own reliability, ordering, and encryption (TLS 1.3 baked in)
- Eliminates the TCP 3-way handshake + TLS handshake latency
- Supports **multiplexing without head-of-line blocking** (a major TCP weakness)

```
HTTP/1.1  →  TCP + TLS (sequential requests, head-of-line blocking)
HTTP/2    →  TCP + TLS (multiplexed, but TCP head-of-line blocking remains)
HTTP/3    →  QUIC (UDP-based, multiplexed, no head-of-line blocking)
```

> **Interview mention:** If asked about optimizing latency for a global service, mentioning HTTP/3/QUIC shows depth.

---

## Common Interview Scenarios

### "Design a video calling app" (Zoom, Google Meet)
→ Use **UDP** for audio/video streams. Latency matters more than perfect delivery.
→ Use **TCP** (or WebSockets over TCP) for signaling (who's calling whom, session setup).

### "Design a chat application" (WhatsApp)
→ Use **TCP** via WebSockets. Messages must be delivered and in order.

### "Design an online multiplayer game"
→ Use **UDP** for real-time position/state updates.
→ Use **TCP** for reliable events (kills, score, inventory changes).

### "Why is your API slow?"
→ Could be TCP handshake latency — consider connection pooling or persistent connections (HTTP keep-alive).

---

## Key Takeaways for Interviews

1. **IP handles addressing and routing** — unreliable by itself.
2. **TCP = reliability** at the cost of latency and overhead. Default choice for most application data.
3. **UDP = speed** at the cost of reliability. Use when you control the application layer and can tolerate or handle loss yourself.
4. Always know **which protocol your chosen tech uses** — Redis uses TCP, DNS uses UDP, video calls use UDP, HTTP uses TCP.
5. **QUIC/HTTP/3** is the modern direction — UDP with custom reliability, significantly lower latency.
6. A server handles many clients simultaneously because each connection is a unique **(src IP, src port, dst IP, dst port)** 4-tuple.

---

## Next Topic → HTTP/HTTPS

Sure! Let me rebuild TCP from scratch with full clarity.

---

## TCP — Think of it like a Certified Mail Service

Imagine you're sending 10 important documents to a friend.

**Without TCP (like UDP):** You just throw all 10 letters into a postbox. Some might get lost. Some might arrive out of order. You have no idea if your friend got them all.

**With TCP:** You number every page (1 of 10, 2 of 10...), your friend signs a receipt for each one, and if a receipt doesn't come back — you resend that page. Your friend also waits until all pages arrive and sorts them in order before reading.

That's TCP. Let's go deeper.

---

## Problem 1: How do two machines agree to talk? → The 3-Way Handshake

Before any data flows, TCP does a "are you there? yes I'm here, okay let's go" exchange.

Think of it like a phone call:

```
You (Client)              Your Friend (Server)
     |                           |
     |   "Hello, can you hear    |
     |-------- SYN ------------->|   me? My number is 100"
     |                           |
     |   "Yes! I hear you (100). |
     |<------ SYN-ACK -----------|   My number is 200"
     |                           |
     |   "Great, I hear you      |
     |-------- ACK ------------->|   (200). Let's talk!"
     |                           |
     |======= Data flows ========|
```

- **SYN** = "I want to connect. I'll start counting my data from sequence number X."
- **SYN-ACK** = "Okay. I'll start counting from sequence number Y. I acknowledge your X."
- **ACK** = "I acknowledge your Y. Connection established."

**Why sequence numbers?** Because data is broken into chunks (packets) and sent separately. Sequence numbers let the receiver put them back in the right order.

---

## Problem 2: What if a packet gets lost? → Retransmission

TCP numbers every byte it sends using **sequence numbers**.

```
Sender sends:
  Packet 1 → bytes 1–1000    (seq = 1)
  Packet 2 → bytes 1001–2000 (seq = 1001)
  Packet 3 → bytes 2001–3000 (seq = 2001)

Receiver replies with ACKs:
  ACK 1001  → "I got up to byte 1000, send me 1001 next"
  ACK 2001  → "I got up to byte 2000, send me 2001 next"
  [silence] → Packet 3 was lost!

Sender waits for ACK... timeout expires → retransmits Packet 3
  Packet 3 → bytes 2001–3000 (resent)

Receiver:
  ACK 3001  → "Got it, all good"
```

The ACK number always means **"I've received everything up to this byte, send me this next."**

---

## Problem 3: What if packets arrive out of order?

The internet doesn't guarantee order. Packet 3 might arrive before Packet 2 because they took different routes.

TCP handles this with a **receive buffer**:

```
Packets arrive:   3, 1, 2   (out of order)

Receiver buffer:
  Slot 1: [Packet 1] ✓  (arrived second)
  Slot 2: [Packet 2] ✓  (arrived third)
  Slot 3: [Packet 3] ✓  (arrived first, held in buffer)

Application reads: 1 → 2 → 3  (correct order, always)
```

The receiver holds out-of-order packets in a buffer and only delivers them to the application **in sequence order**.

---

## Problem 4: What if the sender is too fast? → Flow Control

Imagine a fast server sending data to a slow phone. If the server blasts 1000 packets/second and the phone can only process 100/second, the phone's buffer overflows — packets get dropped.

TCP uses a **receive window** to prevent this:

```
Receiver tells sender: "My buffer has space for 64KB right now"
Sender: sends at most 64KB before waiting for ACK
Receiver processes data, frees up buffer space
Receiver tells sender: "Now I have space for 128KB"
Sender: can now send more
```

The receiver **advertises its window size** in every ACK. The sender never exceeds it.

---

## Problem 5: What if the network itself is congested? → Congestion Control

Flow control protects the **receiver**. Congestion control protects the **network**.

If routers between sender and receiver are overloaded, they start dropping packets. TCP detects this (via dropped packets or delayed ACKs) and slows down.

### Slow Start

TCP doesn't blast at full speed right away. It starts slow and ramps up:

```
Round 1: Send 1 packet  → ACK received → double the window
Round 2: Send 2 packets → ACK received → double the window
Round 3: Send 4 packets → ACK received → double the window
Round 4: Send 8 packets → ...

         ↑ Exponential growth until a threshold, then linear growth
         ↑ If a packet is lost → cut window in half, start again
```

This is why a new connection feels slow at first and gets faster — TCP is probing the network's capacity.

---

## Problem 6: How is the connection closed? → 4-Way Teardown

Either side can initiate closing. It takes 4 steps because both sides need to finish sending independently:

```
Client                        Server
  |                              |
  |-------- FIN ----------------->|  "I'm done sending data"
  |<------- ACK -----------------|  "Got it, but I may still have data to send"
  |                              |  [server finishes sending its data...]
  |<------- FIN -----------------|  "Okay I'm done too"
  |-------- ACK ----------------->|  "Acknowledged, connection closed"
```

This is called **half-close** — each direction is closed independently.

---

## Full Picture: What TCP Does for You

```
Problem                    TCP's Solution
---------------------------+----------------------------------
Packets get lost           | Sequence numbers + retransmission
Packets arrive out of order| Receive buffer + reordering
Receiver is too slow       | Flow control (receive window)
Network is congested       | Congestion control (slow start)
Data is corrupted          | Checksum on every segment
Two sides need to sync up  | 3-way handshake
Connection needs clean end | 4-way teardown
```

---

## Why Does This Matter in System Design?

**1. TCP has overhead — that's a real cost**
- The 3-way handshake = 1 round trip before any data flows
- At 100ms latency, that's 100ms wasted just setting up
- Solution: **connection pooling** (reuse existing connections instead of creating new ones for every request)

**2. Head-of-line blocking**
- If Packet 3 is lost, TCP won't deliver Packet 4 and 5 to the app even if they arrived — it waits for Packet 3 to be retransmitted
- This is a TCP fundamental weakness
- HTTP/2 tried to solve this, but TCP-level blocking remained → HTTP/3 (QUIC over UDP) solves it properly

**3. Stateful connections**
- TCP connections are tied to a specific machine
- If your server crashes mid-connection, the client's connection dies
- This is why load balancers need **sticky sessions** or apps need to handle reconnects gracefully

---

## One-Line Summary

> TCP is a reliable postman that numbers every letter, waits for a signature on each one, resends any that go missing, and never delivers them to you out of order — but all that care takes a little extra time.

---Here's how I broke TCP down into 6 individual problems it solves:

| Problem | TCP's mechanism |
|---|---|
| How to start talking | 3-way handshake (SYN → SYN-ACK → ACK) |
| Lost packets | Sequence numbers + retransmission on timeout |
| Out-of-order arrival | Receive buffer holds and reorders |
| Sender too fast | Flow control via advertised receive window |
| Network congested | Congestion control + slow start |
| How to end cleanly | 4-way teardown (FIN → ACK → FIN → ACK) |

Does this make sense now? Any part you want me to drill further — or shall we move to **Topic 3: HTTP/HTTPS**?

# Phase 1 — Topic 3: HTTP / HTTPS

---

## What is HTTP?

**HTTP (HyperText Transfer Protocol)** is the application-layer protocol used for communication between clients (browsers, apps) and servers on the web.

It defines:
- How a **request** is structured (what the client sends)
- How a **response** is structured (what the server replies)
- What **methods** (verbs) are allowed (GET, POST, etc.)
- What **status codes** mean (200, 404, 500, etc.)

HTTP runs **on top of TCP** — it uses TCP's reliable delivery, but defines what the actual messages look like.

---

## HTTP Request Structure

Every HTTP request has 3 parts:

```
GET /api/users/42 HTTP/1.1          ← Request line (method + path + version)
Host: api.example.com               ← Headers (metadata)
Authorization: Bearer eyJhbGci...   ← Headers (continued)
Content-Type: application/json      ←
                                    ← Blank line (separates headers from body)
{"name": "Raj"}                     ← Body (optional, used in POST/PUT)
```

### Parts explained

| Part | What it contains |
|---|---|
| Request line | HTTP method + URL path + HTTP version |
| Headers | Metadata — auth tokens, content type, cache info, cookies |
| Body | Actual data being sent (for POST, PUT, PATCH) |

---

## HTTP Response Structure

```
HTTP/1.1 200 OK                     ← Status line (version + status code + reason)
Content-Type: application/json      ← Headers
Content-Length: 89                  ←
Cache-Control: max-age=3600         ←
                                    ← Blank line
{"id": 42, "name": "Raj"}           ← Body (the actual response data)
```

---

## HTTP Methods (Verbs)

| Method | Purpose | Has Body? | Idempotent? | Safe? |
|---|---|---|---|---|
| GET | Read/fetch a resource | No | Yes | Yes |
| POST | Create a new resource | Yes | No | No |
| PUT | Replace a resource entirely | Yes | Yes | No |
| PATCH | Partially update a resource | Yes | No | No |
| DELETE | Delete a resource | No | Yes | No |
| HEAD | Like GET but response body is omitted | No | Yes | Yes |
| OPTIONS | Ask what methods are supported | No | Yes | Yes |

- **Idempotent** = calling it multiple times gives the same result (GET /user/1 ten times = same result)
- **Safe** = it doesn't modify any data on the server

> **Interview tip:** POST is not idempotent — submitting a payment form twice charges twice. This is why retry logic must be careful with POST, and why **idempotency keys** exist (Stripe uses them).

---

## HTTP Status Codes

Status codes tell the client what happened. Grouped into 5 families:

### 2xx — Success
| Code | Meaning | When used |
|---|---|---|
| 200 OK | Request succeeded | Standard success |
| 201 Created | Resource was created | After a POST |
| 204 No Content | Success, no body | After a DELETE |

### 3xx — Redirection
| Code | Meaning | When used |
|---|---|---|
| 301 Moved Permanently | Resource moved forever | Old URL → new URL |
| 302 Found | Temporary redirect | Redirect after login |
| 304 Not Modified | Cached version is fresh | Browser cache hit |

### 4xx — Client Errors (your fault)
| Code | Meaning | When used |
|---|---|---|
| 400 Bad Request | Malformed request | Missing required field |
| 401 Unauthorized | Not authenticated | No/invalid token |
| 403 Forbidden | Authenticated but not allowed | No permission |
| 404 Not Found | Resource doesn't exist | Wrong URL |
| 409 Conflict | State conflict | Duplicate entry |
| 429 Too Many Requests | Rate limit exceeded | API throttling |

### 5xx — Server Errors (server's fault)
| Code | Meaning | When used |
|---|---|---|
| 500 Internal Server Error | Generic server crash | Unhandled exception |
| 502 Bad Gateway | Upstream server failed | Load balancer got bad response |
| 503 Service Unavailable | Server overloaded/down | During downtime |
| 504 Gateway Timeout | Upstream server too slow | Timeout from load balancer |

---

## HTTP Headers — The Important Ones

### Request headers
| Header | Purpose | Example |
|---|---|---|
| `Host` | Which domain the request is for | `api.example.com` |
| `Authorization` | Auth credentials | `Bearer <token>` |
| `Content-Type` | Format of request body | `application/json` |
| `Accept` | Format client wants in response | `application/json` |
| `Cookie` | Session cookies | `session_id=abc123` |
| `If-None-Match` | Cache validation (ETag) | `"abc123"` |

### Response headers
| Header | Purpose | Example |
|---|---|---|
| `Content-Type` | Format of response body | `application/json` |
| `Cache-Control` | How long to cache | `max-age=3600` |
| `ETag` | Unique ID of resource version | `"abc123"` |
| `Set-Cookie` | Set a cookie on client | `session_id=xyz` |
| `Location` | Redirect destination | `/login` |
| `X-RateLimit-Remaining` | Requests left in window | `45` |

---

## HTTP Versions — The Evolution

### HTTP/1.0
- One TCP connection per request.
- After each response, connection is closed.
- Very slow — new TCP handshake for every single resource.

### HTTP/1.1 (still widely used)
- **Persistent connections** — TCP connection stays open for multiple requests (`Connection: keep-alive`).
- **Pipelining** — client can send multiple requests without waiting for each response.
- Problem: **Head-of-line blocking** — responses must arrive in order. If request 1 is slow, requests 2 and 3 are stuck waiting.

### HTTP/2 (major improvement)
- **Multiplexing** — multiple requests and responses simultaneously on a single TCP connection, fully interleaved.
- **Header compression (HPACK)** — headers are compressed, reducing overhead.
- **Server push** — server can proactively send resources the client will need (e.g., CSS before client asks for it).
- Problem: TCP-level head-of-line blocking still exists — if one TCP packet is lost, all streams stall.

### HTTP/3 (latest — built on QUIC/UDP)
- Runs on **QUIC** instead of TCP.
- Solves TCP head-of-line blocking — each stream is independent at the transport layer.
- **Faster connection setup** — QUIC combines transport + TLS handshake (0-RTT or 1-RTT).
- Used by Google, Facebook, Cloudflare at scale.

```
Version   | Connection      | Multiplexing | Head-of-line blocking | Speed
----------|-----------------|--------------|----------------------|-------
HTTP/1.0  | New per request | No           | Yes (bad)            | Slowest
HTTP/1.1  | Persistent      | Pipelining   | Yes                  | Okay
HTTP/2    | Single TCP      | Yes          | TCP-level            | Fast
HTTP/3    | QUIC (UDP)      | Yes          | No                   | Fastest
```

---

## What is HTTPS?

**HTTPS = HTTP + TLS (Transport Layer Security)**

TLS is a cryptographic protocol that sits between HTTP and TCP. It provides:

1. **Encryption** — data is encrypted so no one in between can read it.
2. **Authentication** — the client verifies it's talking to the real server (via SSL/TLS certificate).
3. **Integrity** — data cannot be tampered with in transit.

### TLS Handshake (simplified)

```
Client                              Server
  |                                    |
  |------- ClientHello --------------->|  "I support TLS 1.3, here are my cipher suites"
  |                                    |
  |<------ ServerHello + Certificate --|  "Use this cipher. Here's my certificate (public key)"
  |                                    |
  |  [Client verifies cert with CA]    |
  |                                    |
  |------- Key Exchange -------------->|  "Here's my part of the shared secret"
  |                                    |
  |<====== Encrypted data now ========>|  Both sides derive the same symmetric key
```

### SSL Certificate
- Issued by a **Certificate Authority (CA)** — trusted third parties like DigiCert, Let's Encrypt.
- Contains the server's **public key** and domain name.
- Client uses it to verify: "I'm really talking to api.example.com, not an imposter."

### Symmetric vs Asymmetric encryption in TLS
- TLS uses **asymmetric encryption** (public/private key) only during the handshake to exchange a shared key.
- Once the shared key is established, **symmetric encryption** (AES) is used for all data — much faster.

---

## Cookies vs Tokens — How HTTP Stays "Stateful"

HTTP is **stateless** — every request is independent. The server has no memory of previous requests by default.

To maintain sessions (e.g., stay logged in), two approaches:

### Cookies (traditional)
- Server sends `Set-Cookie: session_id=abc123` in response.
- Browser automatically includes `Cookie: session_id=abc123` in every subsequent request.
- Server looks up the session in a **session store (Redis/DB)**.
- **Problem:** Doesn't scale well — every server needs access to the session store.

### JWT Tokens (modern, stateless)
- Server generates a **signed token** containing user info (user ID, roles, expiry).
- Client stores the token and sends it in `Authorization: Bearer <token>` header.
- Server **verifies the signature** — no DB lookup needed.
- **Advantage:** Stateless — any server instance can validate the token.
- **Problem:** Can't be invalidated before expiry (unless you maintain a blocklist).

```
Cookies                    JWT Tokens
-----------                -----------
Stored by browser          Stored by client (localStorage / memory)
Auto-sent by browser       Manually added to header
Server needs session store No server-side storage needed
Easier to invalidate       Hard to invalidate before expiry
Vulnerable to CSRF         Vulnerable to XSS if stored in localStorage
```

---

## Caching in HTTP

HTTP has built-in caching mechanisms to reduce server load and latency.

### Cache-Control header
```
Cache-Control: max-age=3600         → Cache for 1 hour
Cache-Control: no-cache             → Revalidate with server before using cache
Cache-Control: no-store             → Never cache (sensitive data)
Cache-Control: private              → Only browser can cache (not CDN)
Cache-Control: public               → CDNs and proxies can cache too
```

### ETag (Entity Tag)
- Server sends `ETag: "abc123"` with a response (a hash of the content).
- Client caches the response and stores the ETag.
- On next request: `If-None-Match: "abc123"`.
- If content hasn't changed → server replies `304 Not Modified` (no body, saves bandwidth).
- If content changed → server replies `200 OK` with new content + new ETag.

---

## REST vs HTTP

These are often confused:

- **HTTP** is a protocol — it defines how messages are formatted and transmitted.
- **REST** is an architectural style — a set of conventions for how to use HTTP to build APIs.

REST conventions:
- Use HTTP methods correctly (GET to read, POST to create, etc.)
- Resources are identified by URLs (`/users/42`)
- Stateless — no session state on server
- Use HTTP status codes correctly

REST uses HTTP, but HTTP can also be used for non-REST things (file downloads, form submissions, etc.).

---

## Key Takeaways for Interviews

1. **HTTP is stateless** — every request is independent. Sessions are maintained via cookies or tokens.
2. **Always use HTTPS** — never design a system that sends credentials or sensitive data over plain HTTP.
3. **HTTP/2 for most APIs** — multiplexing removes the need for connection pooling at the HTTP level.
4. **HTTP/3 / QUIC** for latency-sensitive global services — mention this to show depth.
5. **Cache-Control and ETags** are how you reduce server load — critical for read-heavy systems.
6. **Status codes matter** — returning 200 for an error is a bug. Interviewers notice if you use them correctly.
7. **Idempotency** — GET, PUT, DELETE are idempotent. POST is not. Design retry logic accordingly.
8. **JWT for stateless auth at scale** — scales horizontally without shared session store.

---

## Next Topic → DNS

# Phase 1 — Topic 4: DNS (Domain Name System)

---

## What is DNS?

**DNS (Domain Name System)** is the internet's phone book.

Computers communicate using **IP addresses** (like `142.250.183.46`), but humans use **domain names** (like `google.com`). DNS is the system that translates domain names into IP addresses.

Without DNS, you'd have to memorize the IP address of every website you visit.

```
You type:        google.com
DNS resolves to: 142.250.183.46
Browser connects to: 142.250.183.46:443
```

---

## Why DNS Matters in System Design

- **Every single request** your app makes starts with a DNS lookup.
- DNS affects **latency** — a slow DNS lookup adds delay before the connection even starts.
- DNS is used for **load balancing** (return different IPs for the same domain).
- DNS is used for **failover** (if one server dies, update DNS to point to another).
- DNS is used for **geo-routing** (return IPs closest to the user's location).

---

## The DNS Hierarchy

DNS is organized as a tree with 4 levels:

```
                        . (Root)
                        |
          ┌─────────────┼─────────────┐
         .com          .org          .in    ← Top-Level Domains (TLD)
          |
     ┌────┴────┐
  google.com  amazon.com              ← Second-Level Domains
      |
  maps.google.com                     ← Subdomains
```

Each level is managed by different authorities:
- **Root (.)** — managed by ICANN, 13 root server clusters worldwide.
- **TLDs (.com, .org, .in)** — managed by registries (Verisign manages .com).
- **Second-level domains (google.com)** — managed by the domain owner.
- **Subdomains (maps.google.com)** — configured by the domain owner.

---

## The DNS Resolution Process (Step by Step)

When you type `maps.google.com` in your browser, here's exactly what happens:

```
Browser → OS Cache → Recursive Resolver → Root NS → TLD NS → Authoritative NS
```

### Step 1: Browser cache
- Browser checks its own DNS cache first.
- If found and not expired → use it, done.

### Step 2: OS cache
- Browser asks the OS (operating system).
- OS checks its local DNS cache (`/etc/hosts` on Linux, Windows host file).
- If found → use it, done.

### Step 3: Recursive Resolver (your ISP or 8.8.8.8)
- OS sends the query to a **Recursive Resolver** — usually provided by your ISP, or a public one like Google (8.8.8.8) or Cloudflare (1.1.1.1).
- The resolver does the heavy lifting of finding the answer.
- If the resolver has it cached → return immediately.

### Step 4: Root Name Server
- Resolver asks a **Root Name Server**: "Who handles .com?"
- Root server replies: "Ask the .com TLD name server at this IP."
- (Root servers don't know the final answer — they just know who to ask next.)

### Step 5: TLD Name Server
- Resolver asks the **.com TLD name server**: "Who handles google.com?"
- TLD server replies: "Ask Google's authoritative name server at this IP."

### Step 6: Authoritative Name Server
- Resolver asks **Google's authoritative name server**: "What is the IP for maps.google.com?"
- Authoritative server replies: "It's 142.250.183.78"
- This is the **final answer** — the authoritative server is the source of truth.

### Step 7: Response returned
- Recursive resolver returns the IP to your browser.
- Resolver caches the result for future queries (based on TTL).
- Browser connects to `142.250.183.78`.

```
Browser
  │
  ├──→ Browser cache? ──→ HIT → done
  │
  ├──→ OS cache? ──→ HIT → done
  │
  ├──→ Recursive Resolver (ISP / 8.8.8.8)
  │         │
  │         ├──→ Resolver cache? ──→ HIT → done
  │         │
  │         ├──→ Root Name Server → "ask .com TLD"
  │         │
  │         ├──→ TLD Name Server (.com) → "ask google's NS"
  │         │
  │         └──→ Authoritative NS (google) → "142.250.183.78" ✓
  │
  └──→ Connect to 142.250.183.78
```

---

## DNS Record Types

DNS isn't just about IP addresses. DNS stores many types of records:

| Record Type | Full Name | Purpose | Example |
|---|---|---|---|
| **A** | Address | Maps domain → IPv4 address | `google.com → 142.250.183.46` |
| **AAAA** | IPv6 Address | Maps domain → IPv6 address | `google.com → 2607:f8b0::200e` |
| **CNAME** | Canonical Name | Maps domain → another domain (alias) | `www.google.com → google.com` |
| **MX** | Mail Exchange | Where to send email for this domain | `google.com → smtp.google.com` |
| **NS** | Name Server | Which servers are authoritative for this domain | `google.com → ns1.google.com` |
| **TXT** | Text | Arbitrary text — used for verification, SPF, DKIM | `"v=spf1 include:google.com"` |
| **PTR** | Pointer | Reverse DNS — IP → domain name | `142.250.183.46 → google.com` |
| **SOA** | Start of Authority | Metadata about the zone (TTL, admin email) | — |
| **SRV** | Service | Specifies host + port for a service | Used in VoIP, gaming |

### Most important for system design:
- **A record** — the core record, domain to IPv4.
- **CNAME** — aliases. `api.example.com` → `example.com`. Very useful for CDNs.
- **MX** — needed for email delivery.
- **TXT** — used for domain ownership verification (Google Search Console, AWS, etc.).

---

## TTL — Time To Live

Every DNS record has a **TTL (Time To Live)** value in seconds. It tells resolvers how long to cache the record before asking again.

```
maps.google.com  300  IN  A  142.250.183.78
                 ↑
                 TTL = 300 seconds = 5 minutes
```

### TTL trade-offs

| Low TTL (e.g., 60s) | High TTL (e.g., 86400s = 1 day) |
|---|---|
| Changes propagate quickly | Changes take a long time to propagate |
| More DNS queries → more load on DNS servers | Fewer queries → less load |
| Good for failover scenarios | Good for stable, rarely-changing records |

> **Interview tip:** Before a planned migration or failover, engineers **lower the TTL** days in advance (e.g., from 1 day to 60 seconds). This ensures that when they flip the DNS record, the change propagates in 60 seconds instead of 24 hours.

---

## DNS and Load Balancing

DNS can distribute traffic across multiple servers — this is called **DNS-based load balancing**.

### Round Robin DNS
Return multiple A records for the same domain. The resolver picks one (usually the first):

```
example.com  A  192.168.1.1
example.com  A  192.168.1.2
example.com  A  192.168.1.3
```

Different clients get different IPs → traffic is spread across 3 servers.

**Limitation:** DNS has no awareness of server health. If 192.168.1.2 crashes, DNS still returns it.

### Geo-DNS / Latency-based routing
Return different IPs based on where the user is:

```
User in India  → example.com resolves to 103.21.45.1  (Mumbai server)
User in US     → example.com resolves to 54.20.11.3   (Virginia server)
User in Europe → example.com resolves to 185.60.10.2  (Frankfurt server)
```

Used by: AWS Route 53, Cloudflare, Akamai.
Huge benefit: routes users to the nearest data center → lower latency.

### Weighted routing
Send X% of traffic to one server, Y% to another:

```
v1.example.com → 90% traffic to stable servers
v2.example.com → 10% traffic to new servers (canary deployment)
```

---

## DNS Caching and Propagation

When you update a DNS record, it doesn't instantly change everywhere. Old records are cached at:
- Recursive resolvers (worldwide)
- OS caches on user machines
- Browser caches

**DNS propagation** = the time it takes for the updated record to be seen everywhere = up to the TTL value of the old record.

This is why DNS changes can take minutes to hours (or up to 48 hours with high TTL) to fully propagate.

---

## DNS in Microservices — Service Discovery

In a microservices system, services need to find each other. DNS is one way to do this:

- Each service registers itself under a known DNS name.
- Other services query DNS to find the IP of the service they need.
- When a service scales up (more instances), DNS returns multiple IPs.

Tools like **Consul**, **Kubernetes CoreDNS**, and **AWS Cloud Map** use DNS for internal service discovery.

```
order-service wants to talk to payment-service
  │
  └──→ DNS lookup: payment-service.internal
       → returns 10.0.1.5, 10.0.1.6, 10.0.1.7 (3 instances)
       → order-service connects to one of them
```

---

## DNS Security

### DNS Spoofing / Cache Poisoning
An attacker tricks a resolver into caching a fake DNS record:
- `bank.com → attacker's IP` instead of the real IP.
- Users visiting `bank.com` land on a fake site.

### DNSSEC (DNS Security Extensions)
- Adds **digital signatures** to DNS records.
- Resolvers can verify the record came from the legitimate source.
- Prevents spoofing and cache poisoning.

### DNS over HTTPS (DoH) / DNS over TLS (DoT)
- Traditional DNS queries are **unencrypted** — anyone can see what domains you're looking up.
- DoH/DoT encrypts DNS queries, protecting privacy.
- Used by Cloudflare (1.1.1.1) and Google (8.8.8.8).

---

## Common Interview Scenarios

### "How do you design for low latency globally?"
→ Use **Geo-DNS** (AWS Route 53 latency-based routing) to route users to the nearest data center.
→ Put servers in multiple regions. DNS returns the closest one.

### "How do you handle failover if a server goes down?"
→ Use a **health-check-aware DNS** service (Route 53, Cloudflare).
→ It monitors server health and automatically removes unhealthy IPs from DNS responses.
→ Keep **TTL low** (60s) so clients pick up the change quickly.

### "How does a CDN work?"
→ CDN providers use **CNAME records** and **Anycast DNS** to route users to the nearest CDN edge node.
→ `static.example.com` CNAME → `example.cdn-provider.com` → resolves to nearest edge server.

---

## Key Takeaways for Interviews

1. **DNS is the starting point** of every network request — latency here affects everything.
2. **TTL controls propagation speed** — lower it before planned changes, raise it for stability.
3. **DNS-based load balancing** is simple but has no health awareness — pair it with a health-check service for failover.
4. **Geo-DNS** is how global systems route users to the nearest region — essential for low latency at scale.
5. **DNS caching happens at multiple levels** — browser, OS, resolver. Changes don't propagate instantly.
6. In **microservices**, DNS is used for internal service discovery — each service has a stable DNS name even as IPs change.
7. Always mention **Route 53 or Cloudflare** when discussing DNS in cloud system design — they support health checks, geo-routing, weighted routing.

---

## Next Topic → REST vs RPC

# Phase 1 — Topic 5: REST vs RPC

---

## The Core Question

When two systems need to talk to each other, they need an **agreed-upon interface**. Two dominant styles exist:

- **REST** (Representational State Transfer) — think in terms of *resources*
- **RPC** (Remote Procedure Call) — think in terms of *actions/functions*

Both use HTTP under the hood (usually), but they model the API very differently.

---

## RPC — Remote Procedure Call

### The idea
RPC makes a call to a remote server **feel like calling a local function** in your code.

You don't think "I'm sending an HTTP request to a server." You think "I'm calling `sendMessage(userId, text)`" — and the RPC framework handles the network part transparently.

### Example

```
// Without RPC — you manually construct HTTP requests
fetch("https://api.example.com/messages/send", {
  method: "POST",
  body: JSON.stringify({ userId: 42, text: "Hello" })
})

// With RPC — looks like a normal function call
messageService.sendMessage(userId=42, text="Hello")
// RPC framework handles serialization + HTTP + deserialization
```

### How RPC works internally

```
Client                              Server
  │                                    │
  │  messageService.sendMessage(...)   │
  │                                    │
  │  [RPC framework serializes args]   │
  │                                    │
  │──── HTTP POST /sendMessage ───────>│
  │     Body: {userId: 42, text: "Hi"} │
  │                                    │  [server deserializes]
  │                                    │  [calls actual function]
  │                                    │  [serializes result]
  │<─── Response: {messageId: 99} ─────│
  │                                    │
  │  [RPC framework deserializes]      │
  │  returns result to caller          │
```

---

## REST — Representational State Transfer

### The idea
REST models your API as a set of **resources** (nouns), and you perform standard operations on them using **HTTP methods** (verbs).

- A resource is identified by a **URL**: `/users/42`, `/messages/99`
- You operate on it using HTTP methods: GET, POST, PUT, DELETE

### Example

```
GET    /messages/99        → fetch message 99
POST   /messages           → create a new message
PUT    /messages/99        → update message 99 entirely
PATCH  /messages/99        → partially update message 99
DELETE /messages/99        → delete message 99
```

REST doesn't care about actions — it cares about **what the resource is** and **what standard operation** you're doing to it.

---

## REST vs RPC — Mental Model Difference

This is the most important thing to understand:

| | REST | RPC |
|---|---|---|
| Think in terms of | **Resources (nouns)** | **Actions (verbs)** |
| URL style | `/users/42` | `/getUser` or `/deleteUser` |
| Operations | Standard HTTP methods (GET, POST, PUT, DELETE) | Custom function names |
| Example | `DELETE /messages/99` | `deleteMessage(99)` |

### REST URL examples (good)
```
GET    /users/42
POST   /orders
DELETE /cart/items/5
GET    /products?category=shoes&sort=price
```

### RPC-style URL examples
```
POST   /getUser
POST   /sendMessage
POST   /deleteOrder
POST   /createPayment
```

Notice: RPC URLs use verbs. REST URLs use nouns. RPC often uses POST for everything.

---

## Types of RPC

RPC has evolved over decades. Different implementations use different formats:

### JSON-RPC
- Simple. Uses JSON over HTTP.
- Every call is a POST to the same endpoint.
- Body contains the method name and params.

```json
POST /rpc
{
  "jsonrpc": "2.0",
  "method": "sendMessage",
  "params": { "userId": 42, "text": "Hello" },
  "id": 1
}
```

### XML-RPC / SOAP
- Older. Uses XML over HTTP.
- Very verbose. Common in enterprise systems (banking, government).
- SOAP adds strict contracts (WSDL files), security, and transactions.

### gRPC (Google RPC) — the modern standard
- Developed by Google. Open source.
- Uses **Protocol Buffers (protobuf)** for serialization — binary format, much smaller and faster than JSON.
- Runs over **HTTP/2** — multiplexing, bidirectional streaming.
- You define your API in a `.proto` file — strongly typed contract.
- Code is **auto-generated** in any language from the `.proto` file.

```protobuf
// message.proto
service MessageService {
  rpc SendMessage (SendMessageRequest) returns (SendMessageResponse);
  rpc GetMessages (GetMessagesRequest) returns (stream Message);  // streaming!
}

message SendMessageRequest {
  int64 user_id = 1;
  string text = 2;
}

message SendMessageResponse {
  int64 message_id = 1;
}
```

---

## REST vs gRPC — Deep Comparison

| Property | REST | gRPC |
|---|---|---|
| Protocol | HTTP/1.1 or HTTP/2 | HTTP/2 only |
| Data format | JSON (text) | Protobuf (binary) |
| Payload size | Larger (verbose JSON) | Smaller (compact binary) |
| Speed | Slower | Faster (3–10x smaller payloads) |
| Streaming | Limited (SSE, WebSockets) | Native bidirectional streaming |
| Type safety | No (unless OpenAPI/Swagger) | Yes (enforced by .proto contract) |
| Browser support | Excellent | Poor (needs grpc-web proxy) |
| Human readable | Yes (JSON is readable) | No (binary is not readable) |
| Code generation | Manual or OpenAPI tools | Automatic from .proto |
| Versioning | URL versioning `/v1/`, `/v2/` | Field numbers in proto (backward compatible) |
| Best for | Public APIs, browser clients | Internal microservice communication |

---

## REST vs RPC — When to Use Which

### Use REST when:
- Building a **public API** that external developers will consume
- Your clients are **web browsers** (REST + JSON is natively supported)
- You want **human-readable** requests (easy to debug with curl/Postman)
- Your domain maps naturally to **CRUD resources** (users, products, orders)
- You need broad compatibility (mobile apps, third-party integrations)

### Use gRPC when:
- **Internal microservice communication** — services talking to each other (not browsers)
- You need **high performance** — low latency, small payloads, high throughput
- You need **streaming** — real-time data, live feeds, bidirectional communication
- You want **strict contracts** — strongly typed, auto-generated client/server code
- Polyglot environments — teams using different languages (Java, Go, Python) all share the same `.proto` file

### Real-world usage
```
Company         External API     Internal services
-------------   ------------     ------------------
Google          REST             gRPC
Netflix         REST             gRPC
Uber            REST             gRPC
Twitter/X       REST             Thrift (their own RPC)
```

---

## GraphQL — A Third Option

Worth knowing because it comes up in interviews.

GraphQL is neither REST nor RPC — it's a **query language for APIs** developed by Facebook.

### The problem it solves
With REST, you often face:
- **Over-fetching** — GET /users/42 returns the full user object but you only need the name.
- **Under-fetching** — you need data from multiple endpoints, so you make multiple requests.

### How GraphQL works
The client specifies **exactly what fields it needs** in a single query:

```graphql
query {
  user(id: 42) {
    name
    email
    posts {
      title
      createdAt
    }
  }
}
```

Server returns exactly that — no more, no less. One request, custom shape.

### REST vs GraphQL vs gRPC

| | REST | GraphQL | gRPC |
|---|---|---|---|
| Who defines the shape? | Server | Client | Server (proto) |
| Over/under fetching | Common | Solved | Not an issue (binary) |
| Browser support | ✓ | ✓ | ✗ (needs proxy) |
| Real-time | Limited | Subscriptions | Native streaming |
| Best for | Simple CRUD APIs | Complex, flexible data fetching | High-perf internal services |
| Used by | Everyone | Facebook, GitHub, Shopify | Google, Netflix, Uber internals |

---

## REST API Design Best Practices

Since REST is the most common in interviews, here are the design rules:

### URL design
```
✓ GET    /users              → list all users
✓ GET    /users/42           → get user 42
✓ POST   /users              → create user
✓ PUT    /users/42           → replace user 42
✓ PATCH  /users/42           → partially update user 42
✓ DELETE /users/42           → delete user 42
✓ GET    /users/42/orders    → get orders for user 42

✗ GET    /getUsers           → verb in URL (RPC style)
✗ POST   /deleteUser/42      → wrong method for delete
✗ GET    /users/42/getOrders → verb in nested resource
```

### Versioning
```
URL versioning (most common):    /v1/users, /v2/users
Header versioning:               Accept: application/vnd.api+json;version=2
Query param versioning:          /users?version=2
```

### Pagination
```
Cursor-based (best for large datasets):
  GET /posts?cursor=abc123&limit=20

Offset-based (simpler):
  GET /posts?page=3&limit=20

Response includes:
{
  "data": [...],
  "next_cursor": "xyz789",
  "has_more": true
}
```

### Filtering, sorting, searching
```
GET /products?category=shoes&color=red&sort=price&order=asc
GET /users?search=raj&role=admin
```

---

## Idempotency in API Design

Critical concept that comes up in payment systems and retry logic.

| Method | Idempotent? | Meaning |
|---|---|---|
| GET | ✓ Yes | Read same data every time |
| DELETE | ✓ Yes | Deleting already-deleted resource = same end state |
| PUT | ✓ Yes | Replacing with same data = same end state |
| POST | ✗ No | Calling twice creates two resources |
| PATCH | ✗ No | Depends on implementation |

### Idempotency keys
For non-idempotent operations (POST), use an **idempotency key**:

```
POST /payments
Idempotency-Key: unique-key-abc123
Body: { amount: 500, currency: "INR" }
```

Server stores the key + result. If the same key arrives again (retry), return the stored result instead of processing again. Stripe uses this exact pattern.

---

## Key Takeaways for Interviews

1. **REST = resources + HTTP verbs.** URLs are nouns. Use GET/POST/PUT/DELETE correctly.
2. **RPC = function calls over network.** gRPC is the modern, high-performance version.
3. **gRPC for internal microservices** (fast, typed, streaming). **REST for public/external APIs** (compatible, human-readable).
4. **GraphQL** solves over/under-fetching — mention it when designing APIs for complex frontend data needs (e.g., mobile apps needing varied data).
5. **Idempotency** — always design payment and write APIs with idempotency in mind. Retries are inevitable in distributed systems.
6. In an interview, when asked "how do your microservices communicate?" — saying gRPC with protobuf shows depth over plain REST.

---

## Next Topic → Latency vs Throughput

# Phase 1 — Topic 6: Latency vs Throughput

---

## The Two Most Important Performance Metrics

When you say a system is "fast" or "slow", you need to be precise. There are two completely different things you could mean:

- **Latency** — how long does ONE request take?
- **Throughput** — how many requests can the system handle per second?

These are independent. A system can have low latency but low throughput, or high throughput but high latency. Understanding both — and the trade-offs between them — is fundamental to system design.

---

## Latency

### Definition
**Latency** is the time from when a request is sent to when the response is received.

Also called: response time, delay, round-trip time (RTT).

```
Client sends request ──────────────────────> Server
                      ←─────────────────────
                      Total time = Latency
```

### Unit
Milliseconds (ms) or microseconds (µs) for very fast systems.

### What contributes to latency?

```
Total Latency = Network latency + Queue wait time + Processing time + DB query time + ...
```

| Component | Typical latency | Notes |
|---|---|---|
| L1 cache read | ~0.5 ns | Fastest possible |
| L2 cache read | ~7 ns | |
| RAM read | ~100 ns | |
| SSD read | ~100 µs | |
| HDD read | ~10 ms | Mechanical, slow |
| Same datacenter network | ~0.5 ms | |
| Cross-region network (India→US) | ~150–200 ms | Speed of light limit |
| DNS lookup (uncached) | ~20–120 ms | |
| TCP handshake | 1 RTT | ~50ms same region |
| TLS handshake | 1–2 RTT | On top of TCP |
| DB query (simple, indexed) | ~1–5 ms | |
| DB query (complex, unindexed) | ~100ms–seconds | |

> **These numbers matter.** Interviewers love when you can reason with actual numbers. Memorize the rough order of magnitude for each.

### Types of latency metrics

Just measuring average latency is misleading. You need percentiles:

| Metric | Meaning |
|---|---|
| p50 (median) | 50% of requests are faster than this |
| p95 | 95% of requests are faster than this |
| p99 | 99% of requests are faster than this |
| p99.9 | 99.9% of requests are faster than this |

### Why percentiles matter more than average

Imagine 100 requests with these latencies (ms):
```
10, 10, 10, 10, 10, 10, 10, 10, 10, 1000
```

- Average = 109ms → looks bad
- p50 = 10ms → most users are fine
- p99 = 1000ms → 1% of users wait 1 second

Now imagine 100 requests:
```
90, 90, 90, 90, 90, 90, 90, 90, 90, 90
```

- Average = 90ms → looks worse than before
- p99 = 90ms → every single user gets consistent experience

The second system is actually better for users despite a higher average. **p99 and p999 catch the "tail latency" that average hides.**

> These slow outliers are called **tail latencies**. They disproportionately affect user experience — especially when a single user request triggers multiple backend calls (if any one is slow, the user feels it).

---

## Throughput

### Definition
**Throughput** is the number of requests (or operations) a system can handle per unit of time.

Also called: capacity, RPS (requests per second), TPS (transactions per second), QPS (queries per second).

```
Throughput = Total requests handled / Time period

Example: 10,000 requests in 1 second = 10,000 RPS
```

### What limits throughput?

Throughput is limited by the **bottleneck** in your system — the slowest/most constrained component:

```
CPU bound    → processor is maxed out (heavy computation)
Memory bound → running out of RAM (caching large datasets)
I/O bound    → waiting on disk reads/writes
Network bound→ saturating network bandwidth
DB bound     → database can't handle more queries
```

To increase throughput, you find and fix the bottleneck. Once you fix it, another component becomes the new bottleneck.

### Bandwidth vs Throughput
These are related but different:

- **Bandwidth** = maximum possible data transfer rate (theoretical ceiling) — like a pipe's diameter
- **Throughput** = actual data transferred — like the actual water flowing through the pipe

Throughput ≤ Bandwidth always.

---

## Latency vs Throughput — The Relationship

They are related but not the same. Here's the key insight:

### Little's Law
```
L = λ × W

L = number of requests in the system
λ = throughput (requests per second)
W = latency (average time per request)
```

Rearranged: **Throughput = Concurrent requests / Latency**

Example:
- You have 100 concurrent users
- Each request takes 100ms (latency)
- Max throughput = 100 / 0.1s = **1000 RPS**

To increase throughput, you can either:
1. Handle more concurrent requests (more servers/threads), OR
2. Reduce latency per request (faster processing)

### The trade-off

**High throughput and low latency are often in tension:**

| Technique | Effect on throughput | Effect on latency |
|---|---|---|
| Batching requests | ↑ Higher (fewer round trips) | ↑ Higher (requests wait to be batched) |
| Caching | ↑ Higher (less DB load) | ↓ Lower (faster responses) |
| Async processing | ↑ Higher (non-blocking) | ↑ Higher (response comes later) |
| Adding more servers | ↑ Higher (parallel handling) | ↓ Lower (less queuing) |
| Compression | ↑ Higher (less bandwidth used) | Slight ↑ (CPU cost to compress) |

> **Batching is the classic trade-off:** Kafka producers batch messages before sending. This increases throughput (fewer network calls) but increases latency (messages wait up to X ms before being sent). You tune the batch size based on which matters more.

---

## Latency Numbers You Must Know

These are the famous "latency numbers every engineer should know" (Jeff Dean, Google):

```
Operation                        Latency
-------------------------------- -----------
L1 cache reference               0.5 ns
Branch mispredict                5 ns
L2 cache reference               7 ns
Mutex lock/unlock                25 ns
Main memory (RAM) reference      100 ns
Compress 1KB with Snappy         3,000 ns  = 3 µs
Send 2KB over 1Gbps network      20,000 ns = 20 µs
SSD random read                  150,000 ns = 150 µs
Read 1MB sequentially from RAM   250,000 ns = 250 µs
Round trip within same datacenter 500,000 ns = 0.5 ms
Read 1MB sequentially from SSD   1,000,000 ns = 1 ms
HDD seek                         10,000,000 ns = 10 ms
Read 1MB sequentially from HDD   20,000,000 ns = 20 ms
Send packet California→Netherlands→California  150,000,000 ns = 150 ms
```

### Key takeaways from these numbers
- **Memory is 1000x faster than SSD, which is 100x faster than HDD** — this is why caching in RAM is so powerful.
- **Network within a datacenter is fast (~0.5ms)**, but cross-region is ~150ms — reason for geo-distributed architecture.
- **Disk I/O is the biggest enemy of low latency** — avoid it in the hot path.

---

## How to Reduce Latency

### 1. Caching
Store results of expensive operations in fast memory (RAM).
```
Without cache: request → DB query (5ms) → response
With cache:    request → Redis lookup (0.5ms) → response
```

### 2. CDN (Content Delivery Network)
Serve static assets from servers geographically close to users.
```
Without CDN: User in Mumbai → Server in US (150ms)
With CDN:    User in Mumbai → CDN edge in Mumbai (5ms)
```

### 3. Connection pooling
Reuse existing TCP/DB connections instead of creating new ones.
```
Without pooling: new TCP handshake per request (+1 RTT overhead)
With pooling:    reuse existing connection (0 overhead)
```

### 4. Avoid N+1 queries
```
Bad:  fetch 100 users, then for each user fetch their profile = 101 DB queries
Good: fetch 100 users + all their profiles in 2 queries (JOIN or batch fetch)
```

### 5. Async I/O / Non-blocking
Don't block a thread while waiting for I/O — serve other requests while waiting.

### 6. Move computation closer to data
Instead of fetching all data to your server and computing there, push computation to the DB (stored procedures, aggregations).

---

## How to Increase Throughput

### 1. Horizontal scaling
Add more server instances behind a load balancer.
```
1 server → handles 1,000 RPS
3 servers → handles 3,000 RPS (roughly)
```

### 2. Async / queue-based processing
Decouple receiving a request from processing it.
```
Request → enqueue to Kafka → immediate response to client
          ↓
     workers process in background (high throughput, higher latency)
```

### 3. Batching
Group multiple operations and process them together.
```
Instead of: 1000 individual DB inserts (1000 round trips)
Do: 1 batch insert of 1000 rows (1 round trip)
```

### 4. Read replicas
Distribute read traffic across multiple DB replicas.

### 5. Sharding
Split data across multiple DB instances — each handles a subset of traffic.

---

## SLA, SLO, SLI — How Latency and Throughput Are Measured in Production

| Term | Full name | Meaning | Example |
|---|---|---|---|
| **SLI** | Service Level Indicator | The actual metric you measure | p99 latency = 120ms |
| **SLO** | Service Level Objective | The target you aim for | p99 latency < 200ms |
| **SLA** | Service Level Agreement | Contract with customers, with penalties | 99.9% of requests < 500ms, or refund |

In system design interviews, when asked about reliability:
> "I'd define an SLO of p99 latency < 200ms and 99.9% availability. I'd instrument the system with distributed tracing (Jaeger) and metrics (Prometheus + Grafana) to track SLIs in real time."

---

## Back-of-Envelope: Latency and Throughput Estimation

This is directly used in Phase 1 Topic 10 (estimation), but let's connect it here.

### Example: How many servers do you need?

```
Given:
  - 10 million daily active users (DAU)
  - Each user makes 10 requests/day on average
  - p99 latency target: 100ms per request
  - Each server handles 500 RPS

Total requests/day = 10M × 10 = 100M requests/day
Peak RPS (assume 10x average during peak):
  Average RPS = 100M / 86400s ≈ 1,160 RPS
  Peak RPS    ≈ 1,160 × 10   = 11,600 RPS

Servers needed = 11,600 / 500 = ~24 servers
(Add 20% buffer → ~30 servers)
```

---

## Key Takeaways for Interviews

1. **Latency = time per request. Throughput = requests per time.** Always clarify which one you're optimizing for — they require different solutions.
2. **Use percentiles (p99, p999), not averages.** Average hides tail latency. Interviewers respect this distinction.
3. **Memorize latency numbers** — RAM vs SSD vs HDD vs network. These come up in back-of-envelope estimation.
4. **Batching trades latency for throughput** — more throughput, but each individual request waits longer.
5. **Caching is the single most effective way to reduce latency** — always propose it in the hot path.
6. **Throughput is limited by the bottleneck.** Identify it first before blindly adding servers.
7. **Little's Law** — throughput = concurrency / latency. Adding servers increases concurrency → increases throughput.
8. Know the **SLI/SLO/SLA** framework — it shows you think about production reliability, not just architecture.

---

## Next Topic → Availability & Reliability

# Phase 1 — Topic 7: Availability & Reliability

---

## What is Availability?

**Availability** is the percentage of time a system is operational and able to serve requests.

```
Availability = Uptime / (Uptime + Downtime) × 100%
```

Example:
```
System was down for 1 hour in a 30-day month
Uptime   = 30 × 24 - 1 = 719 hours
Downtime = 1 hour
Availability = 719 / 720 × 100% = 99.86%
```

---

## The Nines of Availability

"Nines" is the industry shorthand for availability targets:

| Availability | Downtime per year | Downtime per month | Downtime per week |
|---|---|---|---|
| 90% (one nine) | 36.5 days | 72 hours | 16.8 hours |
| 99% (two nines) | 3.65 days | 7.2 hours | 1.68 hours |
| 99.9% (three nines) | 8.76 hours | 43.8 minutes | 10.1 minutes |
| 99.99% (four nines) | 52.6 minutes | 4.38 minutes | 1.01 minutes |
| 99.999% (five nines) | 5.26 minutes | 26.3 seconds | 6.05 seconds |
| 99.9999% (six nines) | 31.5 seconds | 2.6 seconds | 0.6 seconds |

### What level do you target?

| System type | Typical target |
|---|---|
| Internal tools, dashboards | 99% (two nines) |
| Standard web apps, SaaS | 99.9% (three nines) |
| E-commerce, social media | 99.99% (four nines) |
| Banking, payments, healthcare | 99.999% (five nines) |
| Telecom, air traffic control | 99.9999% (six nines) |

> **Interview tip:** Going from 99.9% to 99.99% sounds small but it means reducing downtime from 43 minutes/month to 4 minutes/month. Each additional nine is 10x harder and more expensive to achieve.

---

## What is Reliability?

**Reliability** is the probability that a system performs its intended function correctly over a period of time.

### Availability vs Reliability — the difference

These are related but not the same:

- **Available** = the system is up and responding
- **Reliable** = the system is up, responding, AND giving correct results

A system can be **available but not reliable**:
- Your API returns 200 OK but the data is wrong (bug, data corruption)
- The server is up but silently dropping 1% of writes

A system can be **reliable but not available**:
- Your DB is correct and consistent but it's down for maintenance

> In interviews, reliability is about correctness + availability together. A truly reliable system is one users can **trust** — it's up AND it gives the right answer.

---

## Why Do Systems Go Down?

Understanding failure causes helps you design against them:

| Cause | Examples |
|---|---|
| Hardware failure | Server crashes, disk fails, NIC fails |
| Software bugs | Memory leak, unhandled exception, deadlock |
| Deployment errors | Bad code pushed, config mistake |
| Dependency failure | Third-party API down, DB unreachable |
| Network failure | Packet loss, DNS failure, BGP routing issue |
| Overload | Traffic spike, DDoS attack |
| Data center issues | Power outage, cooling failure, fire |
| Human error | Wrong command, deleted production DB |
| Natural disasters | Flood, earthquake destroying a data center |

---

## Key Concepts for High Availability

### 1. Redundancy — Eliminate Single Points of Failure (SPOF)

A **Single Point of Failure (SPOF)** is any component whose failure brings down the entire system.

```
Bad (SPOF):
Client → [Single Server] → [Single DB]
          ↑ dies = game over

Good (Redundant):
Client → [Load Balancer]
           ├── Server 1
           ├── Server 2  → [Primary DB] ←→ [Replica DB]
           └── Server 3
```

Redundancy means having **backup components** ready to take over when the primary fails.

Types of redundancy:
- **Active-Active** — multiple instances all serving traffic simultaneously. If one dies, others absorb its traffic.
- **Active-Passive (Hot Standby)** — one primary serves traffic, one standby is idle but ready. On failure, standby takes over.
- **Cold Standby** — backup exists but needs time to spin up. Cheaper but slower recovery.

### 2. Failover

**Failover** is the automatic switching to a backup system when the primary fails.

```
Primary DB fails
    ↓
Health check detects failure (within seconds)
    ↓
Failover triggered automatically
    ↓
Replica promoted to primary
    ↓
DNS updated / load balancer routes to new primary
    ↓
System continues with brief interruption
```

Key metrics in failover:
- **RTO (Recovery Time Objective)** — maximum acceptable time to restore service after failure. "We can be down for at most 5 minutes."
- **RPO (Recovery Point Objective)** — maximum acceptable data loss. "We can lose at most 1 minute of data."

```
Failure occurs
    │
    │←─── RPO ───→│←──────── RTO ────────→│
    │             │                        │
  last          failure               service
  backup        detected              restored
```

Lower RTO and RPO = higher availability = higher cost and complexity.

### 3. Replication

Copy data to multiple machines so if one dies, others have the data.

- **Synchronous replication** — primary waits for replica to confirm write before acknowledging client. Zero data loss but higher latency.
- **Asynchronous replication** — primary acknowledges client immediately, replica catches up later. Lower latency but potential data loss on failure.

### 4. Load Balancing

Distribute traffic across multiple instances. If one instance fails, traffic is rerouted to healthy ones.

```
Load Balancer
  ├── health check every 5s → Server 1 ✓
  ├── health check every 5s → Server 2 ✓
  └── health check every 5s → Server 3 ✗ (failed → removed from pool)

Traffic now distributed between Server 1 and Server 2 only
```

### 5. Health Checks

Continuously test if instances are alive and serving correctly:

- **Passive health checks** — load balancer notices a server is returning errors and removes it.
- **Active health checks** — load balancer pings `/health` endpoint every N seconds. No response = remove from pool.

Good `/health` endpoints check not just "is the server alive" but also "can it actually serve traffic" (DB connected, cache reachable, etc.).

### 6. Geographic Distribution (Multi-region)

Deploy your system across multiple data centers in different regions:

```
Users in India  → Asia-Pacific region (Mumbai)
Users in US     → US-East region (Virginia)
Users in Europe → EU region (Frankfurt)

If Mumbai goes down:
  → Route53 health check detects failure
  → DNS updated to route India traffic to Singapore
  → Brief disruption, then back online
```

This protects against entire data center or region outages.

---

## Availability in Distributed Systems — Series vs Parallel

### Components in Series (AND)
If your system requires ALL components to be up, availability **multiplies**:

```
System = Service A AND Service B AND DB

Availability = A_availability × B_availability × DB_availability
             = 99.9% × 99.9% × 99.9%
             = 99.7%

Three 99.9% components in series = only 99.7% system availability!
```

Each component in a chain **reduces** overall availability. This is why minimizing dependencies in the critical path matters.

### Components in Parallel (OR)
If redundant components exist, availability **improves**:

```
System has 2 servers, either can serve traffic:

Probability BOTH fail = (1 - 99.9%) × (1 - 99.9%)
                      = 0.1% × 0.1% = 0.0001%

Availability = 1 - 0.0001% = 99.9999%

Two 99.9% servers in parallel = 99.9999% availability!
```

Parallel redundancy dramatically improves availability.

> **Interview insight:** This math is why "just add more servers" improves availability — redundant parallel components compound. But chained dependencies kill it — minimize the number of required sequential calls.

---

## Fault Tolerance Patterns

### Circuit Breaker
Prevent a failing dependency from cascading failures to your service.

```
Normal state: requests pass through to downstream service

Downstream service starts failing:
  → Circuit breaker counts failures
  → After threshold (e.g., 50% failure rate) → circuit OPENS
  → All requests immediately return error (fast fail) — no waiting for timeout
  → After cooldown period → circuit goes HALF-OPEN
  → One test request sent through
  → If succeeds → circuit CLOSES (normal again)
  → If fails → back to OPEN state
```

Without circuit breaker: your service waits 30s for each timeout → thread pool exhausted → your service also goes down (cascade failure).
With circuit breaker: fail fast in 1ms → threads freed → your service stays up.

### Retry with Exponential Backoff
Don't retry immediately — back off to avoid overwhelming a recovering service:

```
Attempt 1: fails → wait 1s
Attempt 2: fails → wait 2s
Attempt 3: fails → wait 4s
Attempt 4: fails → wait 8s
...
Max retries: 5

Add jitter (random delay) to prevent all clients retrying at the same time
(thundering herd problem)
```

### Timeout
Always set timeouts on outgoing calls. Without a timeout, a hung downstream service holds your thread forever.

```
// Bad — no timeout, can hang forever
response = callPaymentService(request)

// Good — fail fast after 2 seconds
response = callPaymentService(request, timeout=2000ms)
if timeout: return error to client, log for monitoring
```

### Bulkhead
Isolate failures so one part of the system doesn't consume all resources and bring down everything else.

Named after ship bulkheads — compartments that prevent the whole ship from sinking if one section floods.

```
Without bulkhead:
  Slow payment service → exhausts all 100 threads
  → user profile service also can't get threads → also goes down

With bulkhead:
  Payment service: dedicated thread pool of 30 threads
  Profile service: dedicated thread pool of 30 threads
  Other services:  dedicated thread pool of 40 threads

  Payment service slow → only its 30 threads affected
  Profile service continues serving normally
```

### Graceful Degradation
When a non-critical component fails, return a degraded but functional response instead of a full error.

```
Netflix recommendation service is down:
  Bad:  Show error page — user can't watch anything
  Good: Show generic popular movies instead of personalized recommendations
        (degraded experience, but user can still watch)

Twitter trending topics service is down:
  Good: Hide the trending section, show rest of feed normally
```

---

## Measuring Availability

### MTBF and MTTR

| Metric | Full name | Meaning |
|---|---|---|
| **MTBF** | Mean Time Between Failures | Average time system runs before failing. Higher = more reliable. |
| **MTTR** | Mean Time To Recovery | Average time to restore service after failure. Lower = more available. |
| **MTTF** | Mean Time To Failure | For non-repairable systems — average lifetime before permanent failure. |

```
Availability = MTBF / (MTBF + MTTR)

Example:
  MTBF = 1000 hours (fails once every 1000 hours)
  MTTR = 1 hour (takes 1 hour to recover)
  Availability = 1000 / (1000 + 1) = 99.9%

To improve availability:
  → Increase MTBF (make system more reliable — better hardware, testing)
  → Decrease MTTR (recover faster — automation, runbooks, monitoring)
```

**MTTR is often more impactful than MTBF.** A system that fails often but recovers in seconds can have higher availability than a system that rarely fails but takes hours to recover.

---

## Chaos Engineering

**Chaos engineering** is the practice of intentionally injecting failures into a system to test its resilience.

Netflix pioneered this with **Chaos Monkey** — a tool that randomly terminates production servers to ensure the system handles it gracefully.

The idea: if failures are inevitable, find them on your terms (controlled chaos) rather than at 3am during a traffic spike.

Other tools: Gremlin, Chaos Mesh (Kubernetes).

> Mentioning chaos engineering in an interview signals you think about operational resilience, not just architecture.

---

## Common Interview Scenarios

### "How do you design a system with 99.99% availability?"
→ Redundancy at every layer (servers, DB, load balancer)
→ Active-active multi-region deployment
→ Health checks + automatic failover
→ Circuit breakers on all downstream calls
→ Minimize components in the critical path (series = kills availability)
→ Define RTO and RPO targets upfront

### "What happens if your database goes down?"
→ Primary DB fails → replica promoted via automatic failover
→ Application uses circuit breaker to fail fast during transition
→ Queue incoming writes during failover window (if RPO allows)
→ Alert on-call engineer via PagerDuty

### "How do you handle a traffic spike (10x normal)?"
→ Auto-scaling (add instances based on CPU/RPS metrics)
→ Rate limiting (protect DB and downstream services)
→ Graceful degradation (disable non-critical features)
→ Queue-based load leveling (Kafka absorbs spike, workers process at steady rate)

---

## Key Takeaways for Interviews

1. **Know the nines by heart** — 99.9% = 8.7 hours/year downtime, 99.99% = 52 minutes/year.
2. **Availability vs Reliability** — available means up; reliable means up AND correct.
3. **Eliminate SPOFs** — redundancy at every layer.
4. **Series vs parallel math** — chained dependencies kill availability; parallel redundancy boosts it.
5. **RTO and RPO** — always define these when designing for high availability. They drive architecture decisions.
6. **Circuit breaker, retry with backoff, timeout, bulkhead** — the four fault tolerance patterns. Know all of them.
7. **MTTR is often more valuable to reduce than MTBF** — fast recovery > never failing.
8. **Graceful degradation** — partial functionality beats total failure every time.

---

## Next Topic → CAP Theorem

# Phase 1 — Topic 8: CAP Theorem

---

## What is the CAP Theorem?

The **CAP Theorem** (also called Brewer's Theorem, proposed by Eric Brewer in 2000) states:

> **A distributed system can guarantee at most 2 out of 3 properties: Consistency, Availability, and Partition Tolerance.**

You cannot have all three simultaneously. You must choose which two matter most — and that choice shapes your entire database and architecture decision.

---

## The Three Properties

### C — Consistency
Every read receives the **most recent write** or an error.

In other words: all nodes in the distributed system see the **same data at the same time**.

```
Client A writes: x = 10  (to Node 1)
Client B reads:  x       (from Node 2)

Consistent system:   Client B reads x = 10  ✓
Inconsistent system: Client B reads x = 5   ✗ (stale data)
```

> Note: CAP Consistency is NOT the same as ACID Consistency. CAP Consistency is specifically about all nodes agreeing on the same value — it's closer to what ACID calls "linearizability."

### A — Availability
Every request receives a **non-error response** — but it might not be the most recent data.

The system is always up and always responds. It never refuses a request or returns an error due to internal state.

```
Node 2 might not have the latest data yet — but it still responds.
It returns x = 5 (stale) rather than saying "I don't know, error."

Available system:     returns x = 5  (stale but responds)  ✓
Unavailable system:   returns error  "cannot guarantee freshness" ✗
```

### P — Partition Tolerance
The system **continues operating** even when network partitions occur (i.e., some nodes can't communicate with others).

A **network partition** = a network failure that causes some nodes to be cut off from others.

```
Node 1 ──── Node 2 ──── Node 3
              |
              X  ← network partition (Node 3 cut off)
              |
         Node 3 (isolated)
```

Partition tolerance means: even when Node 3 can't talk to Node 1/2, the system keeps working somehow.

---

## The Brutal Truth About P

**In any real distributed system, Partition Tolerance is NOT optional.**

Here's why: network failures are inevitable. Cables get cut, switches fail, AWS availability zones go dark. You cannot prevent partitions — you can only decide what to do when they happen.

So the real choice is:

> **When a partition happens, do you sacrifice Consistency or Availability?**

This makes CAP really a choice between **CP** and **AP**:

```
CP — Consistency + Partition Tolerance
     When a partition occurs → refuse to respond (or return error)
     to avoid returning stale/wrong data

AP — Availability + Partition Tolerance
     When a partition occurs → still respond
     but might return stale data
```

CA (Consistency + Availability, no Partition Tolerance) is only possible in a single-node system with no network — which isn't really "distributed."

---

## Visualizing the Trade-off

```
Normal operation (no partition):
  Node 1 ←──sync──→ Node 2
  Both nodes have same data. Easy — C and A both satisfied.

Network partition occurs:
  Node 1    ✗    Node 2
  (can't communicate)

Now you must choose:

Option 1 (CP): Node 2 refuses to respond
  → Consistent (won't return stale data)
  → NOT available (returns error)

Option 2 (AP): Node 2 responds with what it has (possibly stale)
  → Available (always responds)
  → NOT consistent (might return outdated data)
```

---

## CP Systems — Consistency over Availability

When a partition occurs, CP systems **refuse to answer** rather than risk returning wrong data.

### Characteristics
- Returns an error or timeout if it can't guarantee the data is up-to-date
- All nodes must agree before responding (consensus required)
- Prioritizes correctness over uptime

### Real-world examples

| System | Why CP |
|---|---|
| **HBase** | Strongly consistent reads/writes, sacrifices availability on partition |
| **Zookeeper** | Leader election, distributed locks — must be correct |
| **etcd** | Kubernetes config store — must be consistent |
| **MongoDB** (with strong consistency) | Can be configured CP |
| **Redis** (with WAIT command) | Synchronous replication mode |

### When to choose CP
- **Financial transactions** — bank balance must be correct, not approximate
- **Inventory systems** — you cannot oversell items
- **Leader election / distributed locking** — correctness is critical
- **Configuration management** — wrong config can crash all services

---

## AP Systems — Availability over Consistency

When a partition occurs, AP systems **still respond** with potentially stale data.

### Characteristics
- Always responds, never returns an error due to partition
- Different nodes might have slightly different data temporarily
- Data eventually becomes consistent (**eventual consistency**)

### Real-world examples

| System | Why AP |
|---|---|
| **Cassandra** | Always available, eventual consistency, used at Netflix/Instagram |
| **DynamoDB** (default) | Highly available, eventually consistent reads by default |
| **CouchDB** | AP by design |
| **DNS** | Returns cached (possibly stale) records — always responds |
| **Amazon shopping cart** | You can always add to cart, conflicts resolved later |

### When to choose AP
- **Social media feeds** — seeing a slightly stale post is fine
- **Shopping carts** — availability matters more than perfect sync
- **User sessions / caching** — stale session data is acceptable
- **DNS** — must always respond, even with slightly outdated IPs
- **Metrics and analytics** — approximate counts are fine (likes, views)

---

## Eventual Consistency

AP systems rely on **eventual consistency** — if no new updates are made, all nodes will **eventually** converge to the same value.

```
t=0: Client writes x=10 to Node 1
t=1: Node 2 still has x=5 (hasn't received update yet)
t=2: Node 2 still has x=5
t=3: Replication catches up → Node 2 now has x=10
t=4: All nodes agree: x=10  ← "eventually consistent"
```

The window between t=0 and t=3 is called the **inconsistency window**. How long this lasts depends on replication lag.

### Stronger consistency models (between CP and AP)

Full strong consistency is expensive. There's a spectrum:

```
Weakest ←─────────────────────────────────────────→ Strongest

Eventual     Monotonic     Read-your-    Causal      Linearizable
Consistency  Reads         Writes        Consistency  (Strong)

↑ Higher availability, lower latency     ↑ Lower availability, higher latency
```

| Model | Guarantee |
|---|---|
| **Eventual consistency** | Will converge eventually, no timing guarantee |
| **Monotonic reads** | Once you read a value, you'll never read an older value |
| **Read-your-writes** | You always see your own writes immediately |
| **Causal consistency** | Causally related operations appear in order to all nodes |
| **Linearizability** | All operations appear to execute atomically in real time order |

> **Read-your-writes** is a very practical middle ground — used by many systems. After you post a tweet, you always see it. Others might see it slightly later.

---

## PACELC — The Extension of CAP

CAP only talks about behavior during partitions. But what about during **normal operation** (no partition)?

**PACELC** (proposed by Daniel Abadi) extends CAP:

```
If Partition (P):
  choose between Availability (A) and Consistency (C)

Else (E) — normal operation:
  choose between Latency (L) and Consistency (C)
```

Even without a partition, you face a trade-off:
- **Strong consistency** requires coordination between nodes → higher latency
- **Low latency** means responding immediately without waiting for all nodes → potentially stale

| System | Partition behavior | Normal behavior |
|---|---|---|
| DynamoDB | PA (available) | EL (low latency) |
| Cassandra | PA (available) | EL (low latency) |
| HBase | PC (consistent) | EC (consistent) |
| PostgreSQL | PC (consistent) | EC (consistent) |
| MongoDB | PC (consistent) | EC (consistent) |

> PACELC is more nuanced than CAP. Mentioning it in an interview shows real depth.

---

## CAP in Real System Design Decisions

### "Design a payment system"
→ **CP** — a user's bank balance must be consistent. You cannot show different balances to different parts of the system. A brief error during partition is acceptable; incorrect balance is not.

### "Design Twitter's timeline"
→ **AP** — if your feed is 2 seconds stale, that's fine. The system must always be available. Use eventual consistency.

### "Design a distributed lock / leader election"
→ **CP** — tools like Zookeeper, etcd. The lock must be consistent. Two nodes must not both believe they hold the lock.

### "Design a shopping cart"
→ **AP** — Amazon famously chose AP for shopping carts. You can always add items. Conflicts (e.g., same item added from two devices) are resolved at checkout.

### "Design a social media like counter"
→ **AP** — showing 1M likes vs 1,000,002 likes doesn't matter. Availability and performance matter more. Use eventual consistency.

---

## Common Misconceptions

### Misconception 1: "You can only pick 2 of 3"
Not quite. You can't have all 3 during a partition. During normal operation, you can have all 3. The constraint only applies when partitions occur.

### Misconception 2: "CAP is binary"
In practice, consistency and availability are a spectrum. Systems offer tunable consistency (Cassandra lets you choose consistency level per query: ONE, QUORUM, ALL).

### Misconception 3: "CP means always consistent"
CP means consistent during partitions. If there's no partition, a CP system is also available.

### Misconception 4: "AP means always inconsistent"
AP systems are inconsistent only during partitions. In normal operation, they can be consistent.

---

## Cassandra's Tunable Consistency — Best of Both?

Cassandra is AP by default but lets you tune consistency level per operation:

```
Write/Read consistency levels:
  ONE      → fastest, least consistent (responds after 1 node confirms)
  QUORUM   → majority of nodes must confirm (balance of speed + consistency)
  ALL      → all nodes must confirm (strongest, slowest, least available)
```

With QUORUM reads + QUORUM writes, you get **strong consistency** because:
- You write to majority of nodes
- You read from majority of nodes
- There will always be overlap → you always read the latest write

This is how you get CP behavior from an AP system — at the cost of higher latency.

---

## Key Takeaways for Interviews

1. **P is not optional** in distributed systems — partitions happen. The real choice is CP vs AP.
2. **CP = consistency over availability** — returns error during partition. Use for financial data, locks, configs.
3. **AP = availability over consistency** — returns stale data during partition. Use for social feeds, carts, caches.
4. **Eventual consistency** is the AP trade-off — nodes converge over time.
5. **Read-your-writes** is a practical consistency model — you see your own changes immediately.
6. **PACELC** extends CAP — even without partitions, you trade latency for consistency.
7. **Tunable consistency** (Cassandra) — AP systems can behave like CP with QUORUM reads/writes.
8. Always justify your choice: "I'm choosing AP because slight staleness is acceptable, and availability is critical for user experience" shows interviewer-level thinking.

---

## Quick Reference

```
CAP Property     | Guarantee                          | Sacrifice during partition
-----------------|------------------------------------|---------------------------
Consistency (C)  | All nodes see same data            | Availability
Availability (A) | Always responds (maybe stale)      | Consistency
Partition (P)    | Works despite network failures     | (not optional)

System     | Type | Why
-----------|------|------------------------------------------------
Zookeeper  | CP   | Leader election must be correct
HBase      | CP   | Strong consistency for analytics
Cassandra  | AP   | Always available, eventual consistency
DynamoDB   | AP   | High availability, tunable consistency
DNS        | AP   | Always responds, TTL-based staleness
MongoDB    | CP   | Default strong consistency
Redis      | CP/AP| Depends on replication config
```

---

## Next Topic → ACID vs BASE

# Phase 1 — Topic 9: ACID vs BASE

---

## Why This Topic Matters

ACID and BASE are two opposing philosophies for how databases handle **transactions and consistency**. They map directly onto the CAP theorem discussion:

```
ACID ≈ CP-leaning (consistency-first) → traditional relational databases
BASE ≈ AP-leaning (availability-first) → distributed NoSQL databases
```

Choosing between SQL and NoSQL in an interview almost always comes back to this trade-off.

---

## ACID — The Relational Database Guarantee

ACID is a set of 4 properties that guarantee **reliable transaction processing** — used by relational databases (PostgreSQL, MySQL, Oracle, SQL Server).

### A — Atomicity
A transaction is **all or nothing**. Either every operation in the transaction succeeds, or none do.

```
Transaction: Transfer ₹500 from Account A to Account B

Step 1: Deduct ₹500 from A
Step 2: Add ₹500 to B

If Step 2 fails (e.g., system crash after Step 1):
  → Atomicity guarantees Step 1 is ROLLED BACK
  → Account A still has its original balance
  → No money disappears
```

Without atomicity, you could have money deducted from A but never added to B — money vanishes.

### C — Consistency
A transaction can only bring the database from one **valid state** to another, respecting all defined rules (constraints, triggers, cascades).

```
Rule: account_balance >= 0  (no overdrafts allowed)

Transaction: Withdraw ₹1000 from an account with ₹500

→ Consistency guarantees this transaction is REJECTED
→ Database never enters a state where balance = -500
```

> Note: ACID Consistency is about **respecting database constraints** (foreign keys, unique constraints, check constraints). This is different from CAP Consistency (all nodes seeing the same data).

### I — Isolation
Concurrent transactions don't interfere with each other. Each transaction behaves as if it's the only one running.

```
Transaction A: Read balance (₹1000) → Add ₹100 → Write (₹1100)
Transaction B: Read balance (₹1000) → Add ₹200 → Write (₹1200)

Without isolation (both read ₹1000 at the same time):
  → Final balance = ₹1200 (B's write overwrites A's)
  → ₹100 from Transaction A is LOST

With proper isolation:
  → Transactions are serialized
  → Final balance = ₹1300 (both additions applied correctly)
```

### D — Durability
Once a transaction is committed, it remains committed — even if the system crashes immediately after.

```
Transaction commits → "Payment successful" shown to user
Power outage occurs 1ms later

Durability guarantees:
  → The payment record survives the crash
  → On restart, the data is still there (written to disk, WAL, etc.)
```

Achieved via **Write-Ahead Logs (WAL)** — changes are logged to disk before being applied, so they can be replayed after a crash.

---

## Isolation Levels — The Most Nuanced ACID Property

Isolation has multiple levels, each with different trade-offs between consistency and performance. Higher isolation = more correctness, but more locking = less concurrency.

### The Problems Isolation Levels Solve

| Problem | Description |
|---|---|
| **Dirty Read** | Reading uncommitted data from another transaction (which might get rolled back) |
| **Non-Repeatable Read** | Reading the same row twice in a transaction gives different results (another transaction modified it in between) |
| **Phantom Read** | A query run twice returns different sets of rows (another transaction inserted/deleted rows matching the query) |

### The Four Isolation Levels

```
Level              | Dirty Read | Non-Repeatable Read | Phantom Read | Performance
-------------------|------------|---------------------|--------------|------------
Read Uncommitted   | Possible   | Possible             | Possible     | Fastest
Read Committed     | Prevented  | Possible             | Possible     | Fast
Repeatable Read    | Prevented  | Prevented            | Possible     | Moderate
Serializable       | Prevented  | Prevented            | Prevented    | Slowest
```

#### Read Uncommitted
Can read data that another transaction has written but not yet committed.
```
Transaction A: UPDATE balance SET amount = 1000 (not committed yet)
Transaction B: SELECT amount → reads 1000 (dirty read!)
Transaction A: ROLLBACK
→ Transaction B used a value that never actually existed
```

#### Read Committed (PostgreSQL default)
Only reads committed data. But re-reading the same row within a transaction can give different values if another transaction commits in between.
```
Transaction A: SELECT amount → 500
[Transaction B commits an update: amount = 700]
Transaction A: SELECT amount → 700  (different from first read!)
```

#### Repeatable Read (MySQL InnoDB default)
Guarantees the same row read twice returns the same value within a transaction. But new rows matching a WHERE clause can still appear (phantom reads).
```
Transaction A: SELECT * FROM orders WHERE amount > 100 → 5 rows
[Transaction B inserts a new order with amount = 200, commits]
Transaction A: SELECT * FROM orders WHERE amount > 100 → 6 rows (phantom!)
```

#### Serializable (Strictest)
Transactions behave as if executed one after another (serially), even though they run concurrently. No dirty reads, non-repeatable reads, or phantoms.
```
Guarantees complete isolation — but achieved via heavy locking
or by aborting/retrying transactions that would conflict.
Significant performance cost under high concurrency.
```

> **Interview tip:** Mention that PostgreSQL defaults to Read Committed, MySQL InnoDB defaults to Repeatable Read. Most applications don't need Serializable — it's reserved for financial reconciliation, critical inventory checks, etc.

---

## BASE — The NoSQL Philosophy

BASE stands for:

- **B**asically **A**vailable
- **S**oft state
- **E**ventually consistent

It's the opposite philosophy — prioritize availability and performance, accept temporary inconsistency.

### Basically Available
The system guarantees availability — it will always respond, even if some nodes are down or partitioned (this is the "A" in AP from CAP theorem).

```
Even if 2 out of 5 replica nodes are down,
the system still responds to reads/writes
using the remaining 3 nodes.
```

### Soft State
The state of the system may change over time, even without new input — due to **eventual consistency** processes happening in the background (replication catching up).

```
At t=0: Node A has x=10, Node B has x=5 (stale)
At t=5: Background replication syncs Node B → x=10
        (state changed without new external write)
```

### Eventually Consistent
Given enough time without new writes, all replicas will converge to the same value (as discussed in CAP topic).

```
Write x=10 → propagates to all nodes over time
→ All nodes eventually agree x=10
→ Until then, reads might return stale values
```

---

## ACID vs BASE — Side by Side

| | ACID | BASE |
|---|---|---|
| Priority | Consistency, correctness | Availability, performance |
| Consistency model | Strong (immediate) | Eventual |
| Transactions | Full ACID transactions | Limited or no multi-row transactions |
| Scalability | Harder (vertical scaling common) | Easier (horizontal scaling, sharding) |
| CAP alignment | CP | AP |
| Used by | PostgreSQL, MySQL, Oracle | Cassandra, DynamoDB, MongoDB (default), Couchbase |
| Best for | Banking, inventory, orders | Social media, analytics, IoT, caching |
| Schema | Rigid (predefined schema) | Flexible (schema-less or dynamic) |

---

## Real Database Examples

### Strongly ACID (Relational / SQL)
```
PostgreSQL — full ACID, advanced features (JSONB, full-text search)
MySQL (InnoDB) — full ACID, widely used
Oracle DB — enterprise-grade ACID
SQL Server — Microsoft's ACID-compliant DB
```

### BASE-leaning (NoSQL)
```
Cassandra — AP, eventual consistency, tunable
DynamoDB — AP by default, can opt into strong consistency for single-item reads
MongoDB — ACID for single-document operations, eventual for distributed/sharded
Couchbase — eventual consistency, high availability
Redis — in-memory, can be configured for different durability/consistency
```

### Hybrid: "NewSQL"
Databases that try to give you both ACID guarantees AND horizontal scalability:
```
Google Spanner — globally distributed, strongly consistent, ACID
CockroachDB — distributed SQL, ACID, horizontally scalable
TiDB — MySQL-compatible, distributed, ACID
```

These use sophisticated consensus protocols (like Raft/Paxos) to achieve strong consistency across distributed nodes — at the cost of higher write latency.

---

## When to Choose ACID vs BASE

### Choose ACID (SQL) when:
- **Financial transactions** — money must never be lost or duplicated
- **Inventory management** — can't oversell products
- **Order processing** — order state transitions must be reliable
- **Anything with complex relationships** — foreign keys, joins matter
- Data integrity is more important than raw scale

### Choose BASE (NoSQL) when:
- **Massive scale** — billions of records, horizontal scaling needed
- **High write throughput** — IoT sensor data, logs, analytics events
- **Flexible schema** — data structure varies or evolves frequently
- **Global distribution** — users worldwide need low-latency access
- Slight staleness is acceptable (social feeds, recommendations, view counts)

---

## A Practical Example: E-Commerce System

Most real-world systems use **both** — different databases for different parts:

```
┌─────────────────────────────────────────────────────────┐
│                    E-Commerce Platform                     │
├─────────────────────────────────────────────────────────┤
│                                                             │
│  Orders & Payments        → PostgreSQL (ACID)              │
│  (money must be exact, transactions required)              │
│                                                             │
│  Product Catalog          → MongoDB / Elasticsearch (BASE) │
│  (flexible schema, fast search, eventual consistency OK)   │
│                                                             │
│  Shopping Cart             → Redis / DynamoDB (BASE)        │
│  (high availability, conflicts resolved at checkout)       │
│                                                             │
│  User Activity / Analytics → Cassandra (BASE)               │
│  (massive write volume, approximate is fine)                │
│                                                             │
│  Session Data              → Redis (BASE)                   │
│  (ephemeral, speed matters more than persistence)           │
│                                                             │
└─────────────────────────────────────────────────────────┘
```

This is called **polyglot persistence** — using the right database for each specific need, rather than forcing one database to do everything.

---

## Common Interview Scenarios

### "Why would you use PostgreSQL over MongoDB for a payments system?"
→ ACID transactions guarantee money is never lost or double-spent. Strong consistency (CP) is non-negotiable for financial correctness. MongoDB's eventual consistency in distributed mode could allow temporary discrepancies.

### "Your system needs to handle 1 million writes/second for IoT sensor data — which DB?"
→ Cassandra or a time-series DB (InfluxDB). BASE model — eventual consistency is fine for sensor readings, and the write-optimized architecture (LSM trees) handles massive throughput far better than a traditional RDBMS.

### "How does MongoDB handle ACID if it's NoSQL?"
→ MongoDB supports ACID transactions for single documents always, and since v4.0, multi-document ACID transactions too — but at a performance cost. In sharded/distributed setups, achieving full ACID across shards adds latency, so many teams design schemas to avoid needing multi-document transactions (embed related data in one document).

---

## Key Takeaways for Interviews

1. **ACID = correctness guarantee** (Atomicity, Consistency, Isolation, Durability) — relational DBs, CP-leaning.
2. **BASE = availability guarantee** (Basically Available, Soft state, Eventually consistent) — NoSQL DBs, AP-leaning.
3. **Isolation levels matter** — know Read Committed vs Repeatable Read vs Serializable, and the dirty/non-repeatable/phantom read problems each solves.
4. **NewSQL (Spanner, CockroachDB)** tries to bridge both — strong consistency + horizontal scale, at the cost of write latency.
5. **Polyglot persistence** — real systems use multiple databases, choosing ACID or BASE per component based on what that component needs.
6. Always justify: "I'm using PostgreSQL here because this data requires transactional guarantees" or "I'm using Cassandra here because we need massive write throughput and slight staleness is acceptable."

---

## Quick Reference

```
ACID                          | BASE
-------------------------------|--------------------------------
Atomicity   - all or nothing   | Basically Available - always responds
Consistency - valid states only| Soft State - state changes over time
Isolation   - no interference  | Eventually Consistent - converges over time
Durability  - survives crashes |

CAP alignment: ACID → CP       | BASE → AP

Examples: PostgreSQL, MySQL    | Examples: Cassandra, DynamoDB, MongoDB
```

---

## Next Topic → Back-of-Envelope Estimation

# Phase 1 — Topic 10: Back-of-Envelope Estimation

---

## Why This Matters

Every system design interview eventually asks: **"How many servers/how much storage/how much bandwidth do you need?"**

Back-of-envelope estimation is the skill of taking rough numbers (users, request rates, data sizes) and quickly calculating capacity requirements — **without needing exact precision**. The goal is to be within the right **order of magnitude**, and to demonstrate structured thinking.

This topic ties together everything from Phase 1 — latency numbers, throughput, availability — into one practical skill.

---

## The Numbers You MUST Memorize

### Time conversions
```
1 day    = 86,400 seconds  (~100,000 for quick math)
1 month  = ~2.6 million seconds
1 year   = ~31.5 million seconds (~32 million for quick math)
```

### Data size conversions
```
1 Byte  = 8 bits
1 KB    = 1,000 bytes      (or 1,024 for binary)
1 MB    = 1,000 KB         = 1,000,000 bytes
1 GB    = 1,000 MB         = 10^9 bytes
1 TB    = 1,000 GB         = 10^12 bytes
1 PB    = 1,000 TB         = 10^15 bytes
```

### Common data sizes (estimate these by category)
```
1 character (ASCII)     = 1 byte
1 character (UTF-8)     = 1-4 bytes
1 UUID                  = 16 bytes (36 chars as string)
1 timestamp             = 8 bytes
Tweet (280 chars)       = ~280 bytes (+ metadata ~ a few hundred bytes more)
Average web page        = ~2 MB
Compressed image (JPEG) = 100 KB - 5 MB
1 minute of HD video    = ~50 MB
1 song (MP3)            = ~4-5 MB
```

### Latency numbers (recap from Topic 6)
```
Memory read       = 100 ns
SSD read          = 100 µs
HDD seek          = 10 ms
Same-DC network   = 0.5 ms
Cross-region      = 150 ms
```

---

## The 5-Step Estimation Framework

When asked to estimate, follow this structure every time:

```
Step 1: Clarify assumptions (state them explicitly)
Step 2: Estimate Traffic (requests per second)
Step 3: Estimate Storage (total data + growth rate)
Step 4: Estimate Bandwidth (data in/out per second)
Step 5: Estimate Servers/Memory (based on throughput and cache needs)
```

Always **state your assumptions out loud**. Interviewers care about your reasoning process, not the exact final number.

---

## Step 1: Clarify Assumptions

Before calculating anything, establish baseline numbers. Common ones:

```
Total users (registered)        e.g., 500 million
Daily Active Users (DAU)        e.g., 10% of total = 50 million
Actions per user per day        e.g., 5 posts viewed, 1 post created
Read:Write ratio                 e.g., 100:1 (most systems are read-heavy)
Peak traffic multiplier          e.g., 2-3x average (peak hours)
Data retention period            e.g., 5 years
```

> If the interviewer doesn't give you numbers, **propose reasonable ones and state them**. E.g., "Let's assume 10 million DAU, since that's a reasonable scale for this product."

---

## Step 2: Estimate Traffic (QPS — Queries Per Second)

### Formula
```
Average QPS = (Total daily requests) / 86,400 seconds
Peak QPS    = Average QPS × Peak multiplier (usually 2-3x)
```

### Worked Example: Twitter-like system

```
Given:
  DAU = 100 million
  Each user posts 2 tweets/day, reads 50 tweets/day

Write requests/day = 100M × 2  = 200 million tweets/day
Read requests/day  = 100M × 50 = 5 billion reads/day

Average Write QPS = 200,000,000 / 86,400 ≈ 2,315 QPS
Average Read QPS  = 5,000,000,000 / 86,400 ≈ 57,870 QPS

Peak Write QPS (3x) ≈ 6,945 QPS
Peak Read QPS (3x)  ≈ 173,610 QPS

Read:Write ratio ≈ 25:1
```

> **Key insight:** Read-heavy systems (social media, e-commerce browsing) need read-optimized architecture — caching, read replicas, CDNs. Write-heavy systems (logging, IoT, analytics) need write-optimized architecture — append-only logs, batching, LSM-tree databases.

---

## Step 3: Estimate Storage

### Formula
```
Storage per day = (number of writes/day) × (size per record)
Total storage   = Storage per day × retention period
```

### Worked Example: Twitter-like system

```
Given:
  200 million tweets/day
  Each tweet ≈ 280 chars (text) + metadata (user_id, timestamp, tweet_id, etc.) ≈ 300 bytes total

Storage per day = 200,000,000 × 300 bytes
                = 60,000,000,000 bytes
                = 60 GB/day

Storage per year = 60 GB × 365 ≈ 21.9 TB/year

Storage for 5 years ≈ 110 TB

Now add media (images/videos):
  Assume 10% of tweets have a 200 KB image
  = 20,000,000 images/day × 200 KB
  = 4,000,000,000,000 bytes/day = 4 TB/day (media alone!)

Media storage/year ≈ 1.4 PB/year
```

> **Key insight:** Media/blob storage dwarfs text storage almost always. This is why media goes to **object storage (S3)**, not the database — databases store metadata + URLs/references to the media.

---

## Step 4: Estimate Bandwidth

### Formula
```
Bandwidth = (requests per second) × (average response size)
```

### Worked Example: Twitter-like system

```
Read bandwidth (ingress to users):
  Peak Read QPS = 173,610 QPS
  Average tweet response size ≈ 300 bytes (text) — assume images served via CDN separately

  Bandwidth = 173,610 × 300 bytes ≈ 52 MB/s

Write bandwidth (incoming tweets):
  Peak Write QPS = 6,945 QPS
  Average tweet size ≈ 300 bytes

  Bandwidth = 6,945 × 300 bytes ≈ 2 MB/s
```

> Convert to common units: 52 MB/s ≈ 416 Mbps — well within standard data center network capacity (typically 1-10 Gbps per server).

---

## Step 5: Estimate Servers and Memory

### Application servers

```
Formula:
  Number of servers = Peak QPS / QPS each server can handle

Given:
  Peak Read QPS = 173,610
  Each server handles ~1,000 QPS (reasonable for a typical API server)

  Servers needed = 173,610 / 1,000 ≈ 174 servers
  (add 20-30% buffer for redundancy/failover → ~220 servers)
```

### Cache sizing (for hot data)

```
Formula:
  Cache size = (% of data accessed frequently) × (total data size)

Given:
  Total tweets/day = 60 GB
  Assume "hot" tweets (recent, viral) = top 20% accessed 80% of the time (Pareto principle)

  Hot data per day = 60 GB × 20% = 12 GB
  Cache for last 3 days of hot data ≈ 36 GB

  → A single Redis instance with 64GB RAM could hold this comfortably
  → Or a small Redis cluster for redundancy
```

### Database sizing

```
Given total storage (5 years) = 110 TB (text) + 1.4 PB×5 (media, but media goes to S3 not DB)

  Database (metadata only) ≈ 110 TB
  → Needs sharding. A single DB server might handle 1-5 TB comfortably (with SSDs)
  → Shards needed ≈ 110 TB / 2 TB per shard ≈ 55 shards
```

---

## Full Worked Example: "Design a URL Shortener" (classic interview question)

Let's run through the complete framework for a smaller, classic example.

### Assumptions
```
- 100 million new URLs shortened per month
- Read:Write ratio = 100:1 (URLs are read/redirected much more than created)
- Average URL length = 100 bytes
- Shortened URL = 7 characters
- Data retention = 5 years
```

### Step 2: Traffic
```
Write requests/month = 100,000,000
Write requests/day   = 100,000,000 / 30 ≈ 3.33 million/day
Write QPS (avg)      = 3,330,000 / 86,400 ≈ 38.5 QPS

Read QPS (avg) = 38.5 × 100 = 3,850 QPS
Peak Read QPS (2x) ≈ 7,700 QPS
```

### Step 3: Storage
```
Per URL record: 
  original URL (~100 bytes) + short code (7 bytes) + metadata (created_at, user_id, etc. ~50 bytes)
  ≈ 160 bytes total

URLs created over 5 years = 100M/month × 12 × 5 = 6 billion URLs

Total storage = 6,000,000,000 × 160 bytes
              = 960,000,000,000 bytes
              ≈ 960 GB ≈ ~1 TB
```

### Step 4: Bandwidth
```
Read bandwidth = 7,700 QPS × 160 bytes ≈ 1.2 MB/s  (tiny — easily cacheable)
Write bandwidth = 38.5 QPS × 160 bytes ≈ 6 KB/s (negligible)
```

### Step 5: Servers & Cache
```
Servers: 7,700 QPS / 1,000 QPS per server ≈ 8 servers (+ buffer = ~10)

Cache: 1 TB total data, but "hot" URLs (recently created/popular)
       Assume top 20% accessed → 200 GB hot data
       → Still large; cache the top 1% most-accessed (~10 GB) — fits in a single Redis node
```

### Conclusion
```
This is a small-scale system:
  ~10 application servers
  ~1 TB database (easily fits on a single well-provisioned DB, maybe with read replicas)
  ~10 GB Redis cache for hot URLs
  CDN not strictly necessary, but helps for global redirect latency
```

---

## Quick Reference Card — Numbers to Memorize

```
┌─────────────────────────────────────────────────────────┐
│ TIME                                                       │
│   1 day   ≈ 100,000 seconds (86,400 exactly)               │
│   1 month ≈ 2.6 million seconds                            │
│   1 year  ≈ 32 million seconds                             │
├─────────────────────────────────────────────────────────┤
│ DATA SIZES                                                 │
│   1 KB = 10^3 bytes   1 MB = 10^6   1 GB = 10^9            │
│   1 TB = 10^12        1 PB = 10^15                         │
├─────────────────────────────────────────────────────────┤
│ TYPICAL SERVER CAPACITY                                    │
│   App server: ~1,000 QPS (varies by complexity)            │
│   DB server: ~1-5 TB comfortable storage (SSD)              │
│   Single Redis node: 16-64 GB RAM common                    │
│   Network: 1-10 Gbps per server typical                     │
├─────────────────────────────────────────────────────────┤
│ COMMON MULTIPLIERS                                         │
│   Peak traffic = 2-3x average                               │
│   Replication factor = 3x (for redundancy)                  │
│   Add 20-30% buffer to all server estimates                 │
└─────────────────────────────────────────────────────────┘
```

---

## Common Mistakes to Avoid

1. **Getting lost in precision** — interviewers want order-of-magnitude correctness (is it GB or TB? 1,000 QPS or 100,000 QPS?), not exact numbers.
2. **Forgetting peak traffic** — average QPS is not what breaks your system; peak QPS does.
3. **Forgetting replication** — storage estimates should account for 3x replication factor for durability.
4. **Ignoring media/blob storage** — text data is tiny compared to images/videos. Always ask if media is involved.
5. **Not stating assumptions** — silently picking numbers looks worse than saying "let's assume X, a reasonable estimate for this scale."
6. **Forgetting growth over time** — "storage for 5 years" requires multiplying daily/yearly storage by the retention period.

---

## Key Takeaways for Interviews

1. **Follow the 5-step framework**: Assumptions → Traffic (QPS) → Storage → Bandwidth → Servers/Cache.
2. **Memorize the time and data size conversions** — 86,400 seconds/day, 10^9 bytes/GB, etc.
3. **Always calculate peak QPS**, not just average — systems are provisioned for peak, not average.
4. **Read:Write ratio drives architecture** — read-heavy → caching/CDN/replicas; write-heavy → queues/batching/LSM-trees.
5. **Media storage dominates** — always separate metadata (DB) from media (object storage like S3).
6. **State assumptions explicitly** — this is as important as the math itself; it shows structured thinking.
7. **Round aggressively** — 86,400 ≈ 100,000; this makes mental math fast and the error is negligible at this scale.

---

# Phase 1 Complete! 🎉

You've now covered all 10 foundational topics:
1. ✅ Client-Server Model
2. ✅ IP, TCP, UDP
3. ✅ HTTP/HTTPS
4. ✅ DNS
5. ✅ REST vs RPC
6. ✅ Latency vs Throughput
7. ✅ Availability & Reliability
8. ✅ CAP Theorem
9. ✅ ACID vs BASE
10. ✅ Back-of-Envelope Estimation

---

## Next: Phase 2 — Core Components

Phase 2 covers the building blocks you'll combine in every system design: SQL/NoSQL databases, indexing, caching, CDNs, load balancers, message queues, blob storage, file systems, and API gateways.

Ready to start **Phase 2, Topic 1: SQL Databases**?