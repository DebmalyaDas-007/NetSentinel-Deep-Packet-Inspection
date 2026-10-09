# NetSentinel — Deep Packet Inspection & Network Traffic Analyzer

A C++-based Deep Packet Inspection (DPI) engine for analyzing network traffic captured in PCAP files.

The project parses packets across the Ethernet, IPv4, TCP and UDP layers, tracks network flows using the five-tuple, inspects application-layer information such as HTTP Host headers and TLS SNI, classifies traffic, and applies configurable filtering rules.

The project is being developed as a hands-on exploration of **Computer Networks, C++, Linux, networking protocols, concurrency and cybersecurity**.

---

## 🚀 What This Project Does

At a high level:

```text
              PCAP File
                  │
                  ▼
          ┌───────────────┐
          │ Packet Reader │
          └───────┬───────┘
                  │
                  ▼
          ┌───────────────┐
          │ Packet Parser │
          └───────┬───────┘
                  │
          ┌───────┴────────┐
          ▼                ▼
      TCP / UDP       Flow Tracking
          │                │
          └───────┬────────┘
                  ▼
          ┌───────────────┐
          │ DPI / SNI      │
          │ Inspection     │
          └───────┬────────┘
                  │
                  ▼
          ┌───────────────┐
          │ Rule Engine   │
          └───────┬───────┘
                  │
             ┌────┴────┐
             ▼         ▼
          Forward     Drop
```

The engine works on captured network traffic rather than directly intercepting live traffic.

---

# 🧠 Why I Built This

Most applications communicate through layers of networking protocols, but at the lowest level the system ultimately deals with raw bytes.

This project is an attempt to understand what happens between:

```text
Raw Network Bytes
       ↓
Ethernet
       ↓
IP
       ↓
TCP / UDP
       ↓
TLS / HTTP
       ↓
Application
```

Instead of treating networking as an abstraction, this project explores how packets are actually parsed, associated with connections, inspected and processed.

---

# 🌐 Networking Concepts

The project currently focuses on the following protocol layers:

| Layer | Protocol / Concept |
|---|---|
| Data Link | Ethernet |
| Network | IPv4 |
| Transport | TCP / UDP |
| Application | HTTP / TLS |
| Security | TLS SNI inspection |
| Flow Identification | Five-tuple |

---

# 🔑 Five-Tuple Flow Identification

A network flow is identified using:

```text
Source IP
Destination IP
Source Port
Destination Port
Protocol
```

For example:

```text
192.168.1.100:54321
        ↓
142.250.185.206:443
        TCP
```

The corresponding five-tuple is:

```text
(
    192.168.1.100,
    142.250.185.206,
    54321,
    443,
    TCP
)
```

Packets belonging to the same flow can therefore be associated with the same flow state.

This becomes especially important when moving from a single-threaded implementation to a concurrent architecture.

---

# 🔍 Deep Packet Inspection

The engine does more than inspect source and destination IP addresses.

It attempts to inspect application-layer information contained within packets.

For HTTPS traffic, the project examines the TLS Client Hello and attempts to extract the **Server Name Indication (SNI)**.

For example:

```text
TLS Client Hello
       │
       ▼
   Extensions
       │
       ▼
   SNI Extension
       │
       ▼
"www.example.com"
```

This allows the engine to associate a TLS connection with a hostname without decrypting the actual application payload.

---

# 🧩 Packet Structure

A simplified packet can be viewed as:

```text
┌───────────────────────────────┐
│ Ethernet Header               │
├───────────────────────────────┤
│ IPv4 Header                   │
├───────────────────────────────┤
│ TCP / UDP Header              │
├───────────────────────────────┤
│ Application Payload           │
│                               │
│ TLS / HTTP / other data       │
└───────────────────────────────┘
```

The parser walks through these headers sequentially and extracts the fields required by the DPI engine.

---

# ⚙️ Architecture

The project is structured around several independent components.

```text
PCAP Reader
     │
     ▼
Packet Parser
     │
     ▼
Five-Tuple / Flow Tracker
     │
     ▼
Application Inspection
     │
     ├── HTTP Host
     │
     └── TLS SNI
     │
     ▼
Traffic Classification
     │
     ▼
Rule Engine
     │
     ├── Allow
     │
     └── Block
```

---

# 📁 Project Structure

```text
NetSentinel/
│
├── include/
│   ├── pcap_reader.h
│   ├── packet_parser.h
│   ├── sni_extractor.h
│   ├── types.h
│   ├── rule_manager.h
│   ├── connection_tracker.h
│   ├── load_balancer.h
│   ├── fast_path.h
│   └── thread_safe_queue.h
│
├── src/
│   ├── pcap_reader.cpp
│   ├── packet_parser.cpp
│   ├── sni_extractor.cpp
│   ├── types.cpp
│   ├── main_working.cpp
│   └── dpi_mt.cpp
│
├── generate_test_pcap.py
├── test_dpi.pcap
├── CMakeLists.txt
└── README.md
```

---

# 🧵 Single-Threaded Implementation

The first implementation processes packets sequentially.

```text
PCAP Reader
     │
     ▼
Packet Parser
     │
     ▼
Flow Tracker
     │
     ▼
DPI
     │
     ▼
Rule Engine
     │
     ▼
Output
```

This version is intentionally simpler and acts as the foundation for understanding the complete packet-processing pipeline.

The main processing loop performs:

1. Read packet from PCAP
2. Parse Ethernet/IP/TCP/UDP headers
3. Construct the five-tuple
4. Locate or create the corresponding flow
5. Inspect application-layer information
6. Classify the traffic
7. Apply filtering rules
8. Forward or drop the packet

---

# ⚡ Multi-Threaded Architecture

The next stage introduces parallel packet processing.

```text
                       Reader
                          │
                          ▼
                  ┌──────────────┐
                  │ Load Balancer│
                  └──────┬───────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
           Worker 1   Worker 2   Worker 3
              │          │          │
              └──────────┼──────────┘
                         ▼
                   Output Queue
                         │
                         ▼
                   Output Writer
```

The main idea is to distribute packets between workers while preserving flow locality.

Packets belonging to the same five-tuple should consistently reach the same processing worker.

Conceptually:

```text
hash(five_tuple) % number_of_workers
```

This allows each worker to maintain its own flow state without requiring every flow lookup to be globally synchronized.

---

# 🔄 Producer-Consumer Model

The multi-threaded version uses queues to communicate between processing stages.

```text
Producer
   │
   ▼
┌───────────────┐
│ Thread-Safe   │
│ Queue         │
└───────────────┘
   │
   ▼
Consumer
```

The queue uses synchronization primitives such as:

- `std::mutex`
- `std::condition_variable`
- `std::queue`

The goal is to allow producers and consumers to operate concurrently without corrupting shared state.

---

# 🔐 Rule Engine

The rule engine can be used to classify traffic according to configurable rules.

Possible rule categories include:

```text
IP Address
Domain
Application
Protocol
Port
```

Example:

```text
BLOCK DOMAIN → youtube.com
BLOCK IP     → 192.168.1.50
BLOCK APP    → YouTube
```

The simplified decision process is:

```text
Packet
  │
  ▼
Check Source IP
  │
  ├── Blocked → DROP
  │
  ▼
Check Application
  │
  ├── Blocked → DROP
  │
  ▼
Check Domain
  │
  ├── Blocked → DROP
  │
  ▼
FORWARD
```

---

# 🧪 Example Processing Flow

Consider a packet belonging to:

```text
192.168.1.100:54321
        ↓
142.250.185.206:443
        TCP
```

The engine performs approximately:

```text
1. Read raw packet
       ↓
2. Parse Ethernet header
       ↓
3. Parse IPv4 header
       ↓
4. Parse TCP header
       ↓
5. Construct five-tuple
       ↓
6. Find/create flow
       ↓
7. Inspect TLS Client Hello
       ↓
8. Extract SNI
       ↓
9. Classify application
       ↓
10. Apply rules
       ↓
11. Forward or drop
```

---

# 🛠️ Technologies

### Core

- C++17
- STL
- CMake
- Python

### Networking

- Ethernet
- IPv4
- TCP
- UDP
- HTTP
- TLS
- PCAP

### Systems / Concurrency

- Multithreading
- Mutexes
- Condition variables
- Producer-consumer architecture
- Hash-based work distribution
- Flow state management

### Security

- Deep Packet Inspection
- TLS SNI inspection
- Traffic classification
- Rule-based filtering

### Development Environment

- Linux
- GCC / Clang
- Git
- Wireshark

---

# 🏗️ Building

## Requirements

- Linux/macOS
- C++17 compiler
- CMake
- Python 3

Build using CMake:

```bash
mkdir build
cd build

cmake ..
make
```

---

# ▶️ Running

Example:

```bash
./dpi_engine test_dpi.pcap output.pcap
```

The input is a PCAP capture and the output contains the traffic that was allowed by the filtering rules.

---

# 📊 Planned Improvements

This project is intentionally being developed beyond the initial implementation.

Planned improvements include:

### 1. Live Packet Capture

Move from:

```text
PCAP → DPI
```

towards:

```text
Network Interface
       ↓
Live Capture
       ↓
DPI Engine
```

---

### 2. Network Anomaly Detection

Detect unusual traffic patterns such as:

```text
Traffic Spike
Port Scan
Abnormal Connection Rate
Repeated Connection Failures
Unusual Flow Behaviour
```

Example:

```text
Normal:

10 → 12 → 11 → 14 → 13 connections/sec

Anomaly:

10 → 11 → 13 → 17 → 850 connections/sec
                              ↑
                           ALERT
```

---

### 3. Traffic Statistics

Generate statistics such as:

```text
Total Packets
TCP / UDP Distribution
Top Source IPs
Top Destination IPs
Top Domains
Top Ports
Traffic Volume
Blocked Connections
```

---

### 4. Security Dashboard

A future version will expose traffic statistics through a backend API and visualize them through a web dashboard.

```text
DPI Engine
     │
     ▼
Metrics / Events
     │
     ▼
Backend API
     │
     ▼
Web Dashboard
```

---

### 5. QUIC / HTTP3 Support

Modern HTTPS traffic increasingly uses QUIC over UDP.

Future versions will investigate:

```text
UDP : 443
   ↓
QUIC
   ↓
HTTP/3
```

and how traffic identification changes compared with traditional TCP/TLS traffic.

---

### 6. Performance Benchmarking

The multi-threaded implementation will be benchmarked against the single-threaded implementation using:

- packets/second
- processing latency
- CPU utilization
- memory usage
- throughput
- scaling with worker count

---

# 🎯 What I Am Learning From This Project

This project is being used to strengthen my understanding of:

- Computer Networks
- TCP/IP
- Packet parsing
- Network protocols
- TLS
- Deep Packet Inspection
- Flow tracking
- Hash tables
- C++ memory and data structures
- Multithreading
- Synchronization
- Producer-consumer architecture
- Network security
- Performance optimization
- Linux networking

The goal is not simply to build a packet analyzer, but to understand how a network-security system processes traffic at a low level.

---

# 📚 Learning Path

The project is intentionally approached in stages:

```text
Networking Fundamentals
        ↓
PCAP Format
        ↓
Packet Parsing
        ↓
Five-Tuple / Flow Tracking
        ↓
HTTP / TLS
        ↓
SNI Extraction
        ↓
Rule Engine
        ↓
Concurrency
        ↓
Thread-Safe Queues
        ↓
Parallel Processing
        ↓
Performance Optimization
        ↓
Anomaly Detection
```

---

# ⚠️ Project Status

🚧 **Under Active Development**

The initial implementation is being studied and progressively reimplemented.

The long-term objective is to evolve the project into a high-performance network traffic analysis and security platform with:

- flow-aware DPI
- multithreaded packet processing
- traffic classification
- rule-based filtering
- anomaly detection
- live network monitoring
- security analytics

---

# 👨‍💻 Author

**Debmalya Das**

ECE — National Institute of Technology, Durgapur

Interested in:

```text
Systems
Networking
Cybersecurity
Backend Engineering
Distributed Systems
Artificial Intelligence
```

---

## ⭐ If You Find This Project Interesting

The project is primarily built as a learning exercise to understand what happens underneath high-level networking abstractions.

Rather than treating networking as a black box, the goal is to understand the complete journey:

```text
Bytes
 ↓
Packets
 ↓
Protocols
 ↓
Flows
 ↓
Applications
 ↓
Security Decisions
```