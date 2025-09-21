# Execution Plan for DPDK-GTP-GRO Project

---

## Phase 1 – Environment and Directory Setup

Open a terminal and create the project directory:

```bash
cd ~
mkdir -p dpdk-gtp-gro-app/{src,pcaps,config,logs,scripts,build,docs}
cd dpdk-gtp-gro-app
```

- `src/` → C source code of the project (to be written later).  
- `pcaps/` → Input PCAP files.  
- `config/` → Policy configuration files (e.g., TEID or Inner 5-tuple rules).  
- `logs/` → Logs and test results.  
- `scripts/` → Bash scripts for execution.  
- `build/` → Compiled binaries.  
- `docs/` → Documentation and notes.  

---

## Phase 2 – Adding a PCAP File

Assume the input file is named `input.pcap`. Copy it into the `pcaps/` directory:

```bash
cp /path/to/your/input.pcap pcaps/input.pcap
ls -lh pcaps/
```

---

## Phase 3 – Defining Policy

Create a JSON file in the `config/` directory:

```bash
nano config/policy.json
```

Insert the following initial content (this can be updated later):

```json
{
  "default": "forward",
  "rules": [
    {"teid": 12345, "action": "drop"},
    {"inner": {"src": "10.1.1.1", "dst": "10.2.2.2", "proto": 6}, "action": "count"}
  ]
}
```

Save and exit (`CTRL+O`, then `CTRL+X`).

---

## Phase 4 – Designing the C Code Skeleton

In the `src/` directory, create the main source file `main.c`.

The general structure will be:

- **main**: Parse arguments (pcap input, policy file, mode = none/light/heavy, log directory).  
- **load_pcap()**: Open and read packets from the PCAP file.  
- **parse_gtpu()**: Detect UDP/2152 packets, extract TEID and inner packets.  
- **apply_policy()**: Apply policy rules on TEID or inner 5-tuple.  
- **do_gro_light()**: Call `rte_gro_reassemble_burst()` on a burst of packets.  
- **do_gro_heavy()**: Create context with `rte_gro_ctx_create()`, merge packets with `rte_gro_reassemble()`, and flush with `rte_gro_timeout_flush()`.  
- **write_output()**: Write merged packets or logs into the `logs/` directory.  

⚠️ At this stage only create empty files (implementation will be added later):

```bash
cd src
touch main.c load_pcap.c parse_gtpu.c apply_policy.c do_gro_light.c do_gro_heavy.c write_output.c
```

---

## Phase 5 – Makefile

In the root project directory (`dpdk-gtp-gro-app/`), create a `Makefile`.

Its role: use `pkg-config` and the DPDK toolchain to compile the application and place the binary in `build/`.  
(For now, only create an empty file; content will be added later.)

```bash
touch Makefile
```

---

## Phase 6 – Test Execution Script

In the `scripts/` directory, create a file named `run.sh`:

```bash
nano scripts/run.sh
```

Insert the following content (no C code yet, only execution logic):

```bash
#!/usr/bin/env bash
MODE=${1:-none}   # none | light | heavy
PCAP=${2:-$(pwd)/pcaps/input.pcap}
LOGDIR=$(pwd)/logs
APP=$(pwd)/build/dpdk_gtp_gro_app

mkdir -p "$LOGDIR"
TS=$(date +%Y%m%d-%H%M%S)
LOGFILE="$LOGDIR/run_${MODE}_$TS.log"

sudo $APP   --pcap-input $PCAP   --policy $(pwd)/config/policy.json   --mode $MODE   --log-dir $LOGDIR   2>&1 | tee $LOGFILE
```

Save and exit. Then make it executable:

```bash
chmod +x scripts/run.sh
```

---

## Phase 7 – Running Tests

After writing the C code and building with `make`, run the three modes:

```bash
# Without GRO
./scripts/run.sh none pcaps/input.pcap

# Light GRO
./scripts/run.sh light pcaps/input.pcap

# Heavy GRO
./scripts/run.sh heavy pcaps/input.pcap
```

Each execution stores its logs in the `logs/` directory.

---

## Phase 8 – Analyzing Results

Count the number of output packets:

```bash
tcpdump -r logs/out_light_*.pcap -nn | wc -l
```

Approximate throughput:

```bash
tshark -r logs/out_light_*.pcap -q -z io,stat,1
```

Compare the three modes (`none`, `light`, `heavy`) in terms of throughput and packet count.

---

## Phase 9 – Further Development (Based on Reference Documentation)

- Add flush interval parameter for Heavy mode (e.g., flush = 1, 2, or 4).  
- Simulate multi-queue scenarios as in Case 5 (distribute flows across multiple software queues).  
- Extend to test VxLAN inner flows if suitable PCAP files are available.  
