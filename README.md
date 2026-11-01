# 🌐 QoS: Adaptive Engine for Mixed Home Broadband Traffic

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python Version](https://img.shields.io/badge/Python-3.10%2B-blue.svg)](https://www.python.org/)
[![Enforcement](https://img.shields.io/badge/Enforcement-Linux%20tc%20%7C%20CAKE%20%7C%20FQ__CoDel-green.svg)](https://man7.org/linux/man-pages/man8/tc.8.html)
[![Zero-DPI](https://img.shields.io/badge/Privacy-Zero--DPI%20%2F%20Non--Payload-red.svg)](#-how-it-works)
[![Tata Elxsi](https://img.shields.io/badge/Tata%20Elxsi-TELIPORT%20S4%20Case%20Study%203-orange.svg)](#-overview)

> *An intent-aware, privacy-preserving Quality of Experience (QoE) controller for residential gateways that dynamically eliminates bufferbloat, tracks link capacity, and balances mixed home broadband traffic.*


## 🌟 Highlights

- **🛡️ Zero-DPI Privacy**: Classifies traffic non-invasively using statistical flow metadata (packet timing, burstiness, inter-arrival times) with zero payload decryption.
- **⚡ Bufferbloat Annihilation**: Drops latency under load from **385 ms to 18 ms** (a 95% reduction) using Linux `tc` (HTB + CAKE / FQ-CoDel).
- **📉 Dynamic Capacity Tracking**: Detects sudden WAN bandwidth drops (e.g., 100 Mbps → 20 Mbps) in under **1.5 seconds** via passive RTT gradients and auto-tunes shaper rates.
- **🎯 Intent-Aware User Override**: Prioritizes urgent tasks (e.g., *"Give my work call priority for 45 minutes"*) through an intuitive REST/CLI interface without starving other devices.
- **⚖️ Anti-Starvation Guarantees**: Enforces guaranteed bandwidth floors (10% minimum reservation for bulk transfers), maintaining high fairness (Jain's index > 0.94).
- **🧪 100% Open & Reproducible**: Fully reproducible emulation testbed using Linux network namespaces (`veth`), `netem`, and standard Linux tools.


## ℹ️ Overview

Residential broadband connections are constantly stressed by concurrent competing demands: simultaneous video conferences (Zoom, Teams), competitive gaming, 4K streaming (Netflix, YouTube), and multi-gigabyte OS or game downloads. 

Traditional routers fail because **static priority rules** either starve background downloads or let massive unmanaged queues swell up (**bufferbloat**), destroying real-time call and gaming quality. Furthermore, modern encrypted protocols (QUIC, TLS 1.3, dynamic WebRTC UDP) render legacy port-based packet filters obsolete.

**QoS** is an autonomous edge controller designed for residential gateways. It continuously observes link dynamics, predicts traffic categories statistically, measures real-time WAN capacity, and dynamically shapes queues in the Linux kernel to deliver consistent, predictable Quality of Experience (QoE).

### ✍️ Author
Crafted with ❤️ by **[Yanshi Chauhan](https://github.com/ynsh-ai)** for **Tata Elxsi TELIPORT Season 4 (Case Study 3: Think. Build. Create. Win)**.


## 🚀 Quick Usage

### Start the Adaptive QoS Daemon
Run the autonomous controller on your gateway interface with safe defaults:

```bash
sudo python3 -m src.controller.qos_daemon --interface eth0 --wan-rate 100mbit
```

### Apply a Temporary User Intent (One-liner)
Need priority for an important client call? Trigger a temporary 30-minute priority boost:

```bash
# Prioritize workstation for 30 minutes
curl -X POST http://localhost:8080/api/v1/intent \
  -H "Content-Type: application/json" \
  -d '{"target_ip": "192.168.1.150", "intent": "WORK_CALL", "duration_mins": 30}'
```

### Check Real-Time Telemetry
Monitor queue health, latency, and active flows in your terminal:

```bash
python3 -m src.ui.cli_dashboard
```

```
┌──────────────────────── QoS Engine Live Monitor ────────────────────────┐
│ WAN Rate: 100 Mbps (Est: 98.4 Mbps)  │ Link Status: OPTIMAL             │
│ Active Intent: Work Call (Expires in 24m 12s)                            │
├──────────────────────────────────────┬──────────────────────────────────┤
│ Class                                │ Throughput  │ Delay (p99) │ Loss │
├──────────────────────────────────────┼─────────────┼─────────────┼──────┤
│ 🔵 C1: Interactive (Voice / Call)    │ 1.8 Mbps    │ 16.2 ms     │ 0.0% │
│ 🟢 C2: Gaming (Low Latency UDP)      │ 2.4 Mbps    │ 14.5 ms     │ 0.0% │
│ 🟡 C3: 4K Streaming (Adaptive DASH)  │ 22.0 Mbps   │ 21.0 ms     │ 0.0% │
│ ⚪ C4: Best-Effort Web               │ 8.5 Mbps    │ 24.3 ms     │ 0.0% │
│ 🟣 C5: Bulk Download (ISO / Updates) │ 63.7 Mbps   │ 28.1 ms     │ 0.0% │
└──────────────────────────────────────┴─────────────┴─────────────┴──────┘
```


## ⬇️ Installation

### Prerequisites
- **Operating System**: Linux (Ubuntu 20.04+, Debian 11+, or OpenWrt 22.03+) with kernel $\ge 5.4$.
- **Kernel Modules**: `sch_cake`, `sch_fq_codel`, `sch_htb`, `iproute2`.
- **Python**: Python 3.10 or newer.

### Install in 2 Steps

```bash
# 1. Clone repository
git clone https://github.com/ynsh-ai/QoS.git
cd QoS

# 2. Install dependencies
pip3 install -r requirements.txt
```


## 🏛 How It Works

The engine is decomposed into 5 decoupled, testable modules:

```mermaid
flowchart LR
    LAN["🏠 Home Devices\n(Voice, 4K, Gaming, ISO)"] --> ENFORCE["⚙️ Linux tc\n(HTB + CAKE Leaf Qdiscs)"]
    ENFORCE --> WAN["🌐 WAN / Internet"]

    LAN -.->|Packet Metadata| CLF["🧠 Zero-DPI Classifier\n(Statistical Flow Model)"]
    WAN -.->|Passive RTT / Loss| EST["📡 Link Estimator\n(Kalman / EWMA Filter)"]
    
    INTENT["📱 User Intent API\n('Boost Work Call')"] --> POL["🎯 Policy Governor\n(Dynamic Rate Shaper)"]
    CLF --> POL
    EST --> POL
    POL -->|Atomic Netlink Updates| ENFORCE
```

### Module Breakdown:
1. **Zero-DPI Classifier**: Extracts statistical flow dynamics (packet size distribution, inter-arrival time variance, directional byte ratios) and classifies traffic with confidence scores, allowing instant administrative corrections.
2. **Dynamic Link Capacity Estimator**: Tracks delay gradients ($\frac{dRTT}{dt}$) to detect WAN bottlenecks and down-rate throttling within 1.5 seconds.
3. **Dynamic QoS Policy Engine**: Orchestrates Linux `tc` Hierarchical Token Bucket (HTB) shaping combined with CAKE / FQ-CoDel fair queuing.
4. **Intent Resolver**: Grants temporary priority boosts while enforcing strict fairness ceilings so no other household member is starved.
5. **Fail-Safe Watchdog**: Reverts all kernel traffic control structures to safe-defaults on daemon exit or heartbeat failure.


## 🧪 Benchmark Scenarios & Experiments

Reproduced using the automated namespace testbed (`scripts/setup_testbed.py`):

| Scenario | Challenge | Unmanaged FIFO (Bufferbloat) | Static Priority | **Adaptive QoS (Ours)** |
| :--- | :--- | :--- | :--- | :--- |
| **1. 10 GB ISO + Video Call** | Saturated uplink during Zoom call | ❌ $385\text{ ms}$ latency, $7.8\%$ loss | ⚠️ Call okay, download starved ($0\text{ Mbps}$) | ✅ **$18\text{ ms}$ latency, $78\text{ Mbps}$ download** |
| **2. Sudden WAN Drop** | WAN throttles from $100 \rightarrow 20\text{ Mbps}$ | ❌ $520\text{ ms}$ queue inflation | ❌ Stale rate builds deep buffers | ✅ **Adapts in $1.2\text{ s}$; latency stays $<24\text{ ms}$** |
| **3. 3x 4K Streams + Gaming** | High throughput vs. low-latency | ❌ $210\text{ ms}$ game tick latency | ⚠️ Stream stalls or gaming lag | ✅ **Gaming $<16\text{ ms}$, zero video re-buffering** |

### Run the Automated Benchmarks:
```bash
# Run all benchmark scenarios in isolated network namespaces
sudo python3 tests/run_experiment.py --scenario all --output-dir results/
```


## 🔬 Prototype vs. Production Boundary

| Feature | Reference Prototype | Production Gateway Solution |
| :--- | :--- | :--- |
| **Data Plane** | Linux `tc` (HTB + CAKE) via Netlink | XDP / eBPF kernel offload or hardware DPU (Broadcom/Qualcomm) |
| **Control Plane** | Python 3 daemon + FastAPI REST API | C/Rust embedded daemon integrated with OpenSync / RDK-B |
| **Interfaces** | Virtual Ethernet (`veth`) + `tc netem` | Physical GPON/DOCSIS WAN + Wi-Fi 6/7 EasyMesh (WMM QoS) |
| **Management** | Local REST API + Terminal Dashboard | Cloud Orchestrator via TR-181 / USP standards |


## 💭 Feedback and Contributing

Contributions, feedback, and questions are warmly welcome!
- 💡 Have an idea or question? Open a [Discussion](https://github.com/ynsh-ai/QoS/discussions).
- 🐛 Found a bug? File an [Issue](https://github.com/ynsh-ai/QoS/issues).
- 🤝 Want to contribute? Check out our guidelines and submit a PR!


## 📄 License

This project is licensed under the [MIT License](LICENSE).
