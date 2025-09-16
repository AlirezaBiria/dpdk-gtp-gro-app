# dpdk-gtp-gro-app

A DPDK-based app to parse GTPv1-U traffic, decapsulate inner packets, apply TEID/inner-5tuple policies, and evaluate GRO (light vs heavy).

## Build
TBD (DPDK toolchain, pkg-config)

## Run
See `scripts/run_test.sh`.

## Layout
- `src/` C sources
- `pcap/` test pcaps
- `docs/` design notes & references
- `scripts/` helper scripts
