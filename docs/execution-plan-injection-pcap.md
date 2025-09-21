# Execution Plan for DPDK-GTP-GRO Project (with PCAP Injection on NIC)

## Phase 1 – Environment and Directory Setup

Open a terminal.  
Create and enter the project directory:

```bash
cd ~
mkdir -p dpdk-gtp-gro-app/{src,pcaps,config,logs,scripts,build,docs}
cd dpdk-gtp-gro-app
```

- `src/` → C source code of the project (to be implemented later).  
- `pcaps/` → Input PCAP files.  
- `config/` → Policy files (e.g., TEID or inner five-tuple rules).  
- `logs/` → Logs and test outputs.  
- `scripts/` → Bash scripts for execution.  
- `build/` → Compiled application binary.  
- `docs/` → Documentation and notes.

## Phase 2 – Adding the PCAP File

Assume the input file is named `input.pcap`. Copy it into the project directory:

```bash
cp /path/to/your/input.pcap pcaps/input.pcap
ls -lh pcaps/
```

## Phase 3 – Defining Policy

Create a JSON policy file under `config/`:

```bash
nano config/policy.json
```

Insert the following initial content (this may be modified later):

```json
{
  "default": "forward",
  "rules": [
    {"teid": 12345, "action": "drop"},
    {"inner": {"src": "10.1.1.1", "dst": "10.2.2.2", "proto": 6}, "action": "count"}
  ]
}
```

Save and exit (CTRL+O, then CTRL+X).

## Phase 4 – C Code Skeleton Design

In the `src/` directory, create the primary source file `main.c`.  
(At this stage create files only; implementations will be added later.)

Required components and functions:

- `main`: Parse application arguments (policy file, mode = none/light/heavy, log-dir, and in the NIC scenario also port/queue parameters).
- `nic_init()`: Initialize DPDK EAL and configure RX/TX queues on the receiver NIC.
- `rx_loop()`: Packet receive loop from the NIC (replaces file-based input).
- `parse_gtpu()`: Detect UDP/2152 packets, extract TEID and inner packet.
- `apply_policy()`: Apply policy rules based on TEID or inner five-tuple.
- `do_gro_light()`: Invoke `rte_gro_reassemble_burst()` on bursts of packets.
- `do_gro_heavy()`: Use `rte_gro_ctx_create()` + `rte_gro_reassemble()` + `rte_gro_timeout_flush()` with configurable flush parameter.
- `write_output()`: Record logs and/or output PCAPs to `logs/`.

Create the empty source files:

```bash
cd src
touch main.c nic_init.c rx_loop.c parse_gtpu.c apply_policy.c do_gro_light.c do_gro_heavy.c write_output.c
cd ..
```

Note: In this scenario there is **no** `load_pcap()` function because input is received from the NIC.

## Phase 5 – Makefile

In the project root (`dpdk-gtp-gro-app/`) create a `Makefile`.  
Purpose: use `pkg-config` and the DPDK toolchain to compile the application and place the binary in `build/`.  
(For now create an empty file; content will be added later.)

```bash
touch Makefile
```

## Phase 6 – Test Execution Scripts (PCAP Injection)

This scenario uses two scripts:

- One script to run the DPDK application on the receiver NIC.  
- One script to prepare the sender namespace and inject the PCAP via `tcpreplay` on the sender NIC.

### 6.1) Script to run the DPDK application on the receiver NIC

Create:

```bash
nano scripts/run_app.sh
```

Content:

```bash
#!/usr/bin/env bash
# Run the application on the NIC (the application receives packets from NIC; no PCAP file input)
MODE=${1:-light}   # none | light | heavy
PORT_ID=${2:-0}    # DPDK port identifier (typically 0)
LOGDIR=$(pwd)/logs
APP=$(pwd)/build/dpdk_gtp_gro_app

mkdir -p "$LOGDIR"
TS=$(date +%Y%m%d-%H%M%S)
LOGFILE="$LOGDIR/run_${MODE}_$TS.log"

# Example EAL settings: adjust core mask and memory channels according to your system
# Note: in NIC scenario the application must initialize the port/queues itself (nic_init)
sudo "$APP" -l 2-4 -n 4 --   --port-id $PORT_ID   --policy $(pwd)/config/policy.json   --mode $MODE   --log-dir $LOGDIR   2>&1 | tee "$LOGFILE"
```

Make executable:

```bash
chmod +x scripts/run_app.sh
```

### 6.2) Script to prepare the sender namespace and inject the PCAP

Create:

```bash
nano scripts/run_tcpreplay.sh
```

Content:

```bash
#!/usr/bin/env bash
# Usage: ./scripts/run_tcpreplay.sh <tx_iface_in_kernel> <pcap> [loop] [rate]
# Example: ./scripts/run_tcpreplay.sh enp1s0 pcaps/input.pcap 0 topspeed
TX_IFACE=$1
PCAP=${2:-$(pwd)/pcaps/input.pcap}
LOOP=${3:-0}          # 0 = infinite
RATE=${4:-topspeed}   # topspeed | mbps=200 | pps=100000

set -e
sudo ip netns del txns 2>/dev/null || true
sudo ip netns add txns
sudo ip link set "$TX_IFACE" netns txns
sudo ip netns exec txns ip addr add 10.10.10.1/24 dev "$TX_IFACE" || true
sudo ip netns exec txns ip link set "$TX_IFACE" up
sudo ip netns exec txns ethtool -K "$TX_IFACE" tso on || true

if [[ "$RATE" == topspeed ]]; then
  sudo ip netns exec txns tcpreplay --intf1="$TX_IFACE" --loop="$LOOP" --preload-pcap --topspeed "$PCAP"
elif [[ "$RATE" == mbps* ]]; then
  MBPS=${RATE#mbps=}
  sudo ip netns exec txns tcpreplay --intf1="$TX_IFACE" --loop="$LOOP" --preload-pcap --mbps="$MBPS" "$PCAP"
elif [[ "$RATE" == pps* ]]; then
  PPS=${RATE#pps=}
  sudo ip netns exec txns tcpreplay --intf1="$TX_IFACE" --loop="$LOOP" --preload-pcap --pps="$PPS" "$PCAP"
else
  # No rate limiting and no preload
  sudo ip netns exec txns tcpreplay --intf1="$TX_IFACE" --loop="$LOOP" "$PCAP"
fi
```

Make executable:

```bash
chmod +x scripts/run_tcpreplay.sh
```

Important note: The receiver NIC (where the DPDK application listens) must be bound to `vfio-pci` or `igb_uio`; the sender NIC must remain kernel-owned so `tcpreplay` can transmit.

## Phase 7 – Running the Three Test Modes (with PCAP Injection)

1. Bind the receiver NIC to DPDK (do this once per boot; replace `<PCI_ADDR>` with the actual PCI address of the receiver NIC):

```bash
cd ~/dpdk-gtp-gro-app
sudo ./usertools/dpdk-devbind.py --status
sudo ./usertools/dpdk-devbind.py -b vfio-pci <PCI_ADDR>
sudo ./usertools/dpdk-devbind.py --status
```

2. Run the application on the receiver NIC (open Terminal #1). Execute the three modes:

```bash
# Without GRO
./scripts/run_app.sh none 0

# Light GRO
./scripts/run_app.sh light 0

# Heavy GRO (map the flush parameter in the app as documented; e.g., --gro-flush=2/4)
./scripts/run_app.sh heavy 0
```

3. Inject the PCAP using `tcpreplay` from the sender NIC (open Terminal #2 — the sender NIC remains kernel-owned):

```bash
# Infinite loop, maximum speed
./scripts/run_tcpreplay.sh enp1s0 pcaps/input.pcap 0 topspeed

# Or loop 10 times at 200 Mbps
# ./scripts/run_tcpreplay.sh enp1s0 pcaps/input.pcap 10 mbps=200
```

If you use `testpmd` for comparison, apply the `set fwd csum`, `set port 0 gro on`, and `set gro flush N` commands as described in the DPDK documentation.

All outputs for each run are stored under the `logs/` directory.

## Phase 8 – Results Analysis

If the application produces an output PCAP, count the output packets:

```bash
tcpdump -r logs/out_light_*.pcap -nn | wc -l
```

Approximate throughput:

```bash
tshark -r logs/out_light_*.pcap -q -z io,stat,1
```

Compare the three modes (`none`, `light`, `heavy`) in terms of throughput and packet count.

If the application does not write output PCAPs and only logs statistics, review the `logs/run_*.log` files (e.g., `tail -f logs/run_*.log`) to obtain the relevant metrics.

## Phase 9 – Further Development (Based on Reference Documentation)

- Add a flush-interval parameter in the application for Heavy mode (e.g., `--gro-flush=1|2|4`), mapping directly to `rte_gro_timeout_flush()` or implementing an appropriate flushing policy.  
- Simulate multi-queue operation similar to Case 5 (support `--rxq`, `--txq` arguments and map flows to queues using a hash function; enable multiple RX/TX queues in the NIC configuration).  
- Test VxLAN inner flows if suitable PCAP files are available (when using `testpmd` for comparison, enable tunnel parsing and outer-IP checksum handling as required).

## Vital Notes (Injection Scenario)

- The sender NIC must remain kernel-owned and `tcpreplay` must run on that interface.  
- The receiver NIC must be bound to DPDK and the application must receive packets from that NIC.  
- Loop behavior:
  - `--loop=N` replays the PCAP N times; `--loop=0` replays indefinitely.  
  - For continuous high-rate replay, use `--preload-pcap` together with `--topspeed` or `--mbps=...`.
