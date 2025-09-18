# dpdk-gtp-gro-app

## Introduction

In mobile networks (3G/4G/5G), user traffic is not transmitted directly; instead, packets are encapsulated within a tunneling protocol called **GTP-U (GPRS Tunneling Protocol – User plane)**. By using GTP-U, user packets are carried through separate sessions between network elements such as eNodeB, SGW, PGW, or in 5G architectures, gNB and UPF. This encapsulation enables mobility and session management but introduces additional processing overhead on servers responsible for forwarding and classifying traffic.

To handle these challenges efficiently, we leverage **DPDK (Data Plane Development Kit)**, a framework designed for high-performance packet processing in user space. In this project, DPDK is used to capture GTP-U packets, decapsulate inner traffic, and apply **Generic Receive Offload (GRO)** mechanisms to reduce CPU load and improve throughput. The project evaluates and compares the two main GRO strategies: **Light** and **Heavy**.

---

### Structure of Input Packets

A GTP-U v1 packet carried over UDP/2152 consists of multiple headers stacked before the actual user payload:

+-------------------+
| Ethernet Header   |  ← MAC source/destination
+-------------------+
| Outer IP Header   |  ← IP of mobile network nodes (e.g., eNodeB, UPF)
+-------------------+
| UDP Header        |  ← destination port 2152
+-------------------+
| GTP-U Header v1   |  ← includes TEID (Tunnel Endpoint ID) and control fields
+-------------------+
| Inner IP Header   |  ← real user IP address
+-------------------+
| Inner TCP/UDP     |  ← real transport protocol (e.g., TCP/80, UDP/53)
+-------------------+
| Payload           |  ← user data (e.g., HTTP, video, DNS)
+-------------------+

This structure shows that the actual user packet lies inside an extra tunneling layer and requires decapsulation before further processing.

---

### Role of TEID and Inner Five-Tuple

- **TEID (Tunnel Endpoint Identifier):**  
  Present in the GTP-U header, it identifies the tunnel/session and determines which user or flow the packet belongs to. TEID acts as the outer key for tunnel identification.

- **Inner Five-Tuple:**  
  After decapsulation, the real user packet is extracted. The flow is identified using the five-tuple:
  - srcIP, dstIP, srcPort, dstPort, protocol  
  This inner key is crucial for **GRO**, since GRO merges packets that belong to the same real user flow.

👉 Workflow: first identify the tunnel using TEID, then extract the inner packet, and finally perform GRO based on the inner five-tuple.

---

### DPDK in This Project

DPDK provides the necessary libraries and drivers to:
- Receive packets from NICs or PCAP files.  
- Identify GTP-U packets carried over UDP/2152.  
- Perform decapsulation to access the inner user traffic.  
- Use **librte_gro** to merge packets and reduce per-packet processing overhead.  

---

### GRO and Its Importance

**Generic Receive Offload (GRO)** reduces CPU load by merging multiple consecutive packets of the same flow into larger packets.  

Normally, each incoming packet requires a full processing cycle: mbuf allocation, header parsing, statistics updates, and further pipeline stages. When millions of packets per second arrive, this per-packet overhead dominates CPU usage. GRO addresses this by reducing the total number of mbufs to be processed. Instead of processing N small packets, GRO enables processing only N/k larger packets, where k is the average merge factor.

The CPU cost can be modeled as:

CPU_Cost_no_GRO ≈ N × C_pkt  
CPU_Cost_with_GRO ≈ (N / k) × C_pkt  

where:  
- N = number of input packets,  
- k = average number of packets merged together,  
- C_pkt = base cost of processing one packet.  

For example, if on average 10 packets are merged (k=10), the CPU overhead is reduced by roughly an order of magnitude.

**Two main strategies exist:**  
- **Light GRO:** lightweight, minimal state, low latency; fewer packets merged.  
- **Heavy GRO:** maintains larger state and waits longer to merge more packets; improves throughput but increases memory usage and latency.

In both cases, GRO is applied *after decapsulation* and operates on inner five-tuple flows.

---

### Project Objectives

The primary goals of this project are:
1. Use a PCAP trace containing **GTPv1-U** traffic as input.  
2. Capture and parse packets using DPDK.  
3. Decapsulate GTP headers and extract inner packets.  
4. Apply GRO to the inner flows.  
5. Compare **Light vs Heavy GRO** in terms of merge ratio, throughput, CPU utilization, and latency.

---

### Benefits

- Gain practical understanding of handling GTP-U traffic with DPDK.  
- Evaluate the effectiveness of GRO in tunneled environments.  
- Determine when Light GRO (latency-sensitive scenarios) or Heavy GRO (throughput-sensitive scenarios) should be applied.  
- Provide a reproducible framework for further research in **high-performance packet processing**.  

---

### Summary

This project develops a DPDK-based experimental application that decapsulates tunneled GTP-U packets, applies GRO on inner flows, and evaluates the trade-offs between Light and Heavy GRO. The results will highlight the strengths and weaknesses of each method in balancing throughput and latency for mobile network traffic.

---

## Build
TBD (DPDK toolchain, pkg-config)

## Run
See `scripts/run_test.sh`.

## Layout
- `src/` C sources
- `pcap/` test pcaps
- `docs/` design notes & references
- `scripts/` helper scripts
