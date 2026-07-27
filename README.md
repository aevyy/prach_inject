# prach-inject

A 5G NR PRACH (Msg1) transmitter and flooding tool for O-RAN security
research. `prach-inject` synchronizes to a live gNB over the air, then injects
one or many RACH preambles at the correct Random Access Occasion (RO), each
of which the gNB detects and answers with a Random Access Response — driving
it to allocate a Temporary C-RNTI and scheduler state for a UE that never
completes contention resolution.

It is intended solely for controlled security research on infrastructure you
own or are authorized to test.

## What it does

The tool reads the target cell's parameters, times itself to the gNB's frame
clock via the SSB, and transmits a Zadoff-Chu PRACH preamble so that it lands
in a valid RO. In flood mode it superimposes up to 64 distinct preamble
indices into a single transmission, so one RO carries many apparent
simultaneous access attempts.

The pipeline (see `main.cpp`) runs in ordered steps:

1. **Cell config from InfluxDB** — the cell's PRACH parameters (root sequence,
   ZCZ config, `prach-ConfigurationIndex`, frequency offset, PRB count, ARFCNs)
   are pulled from an InfluxDB bucket populated by a companion sniffer
   (`sni5gect`). InfluxDB is the sole configuration source; there is no manual
   cell entry. See `influx_reader.cpp`.
2. **RO resolution** — `cell_config::resolve_prach_ro()` maps the config index
   to `{format, x, y, subframe}` per TS 38.211 Table 6.3.3.2-2. An occasion
   exists when `SFN % x == y` and the subframe matches (`ro.cpp`).
3. **RA-RNTI computation** — derived from the RO's symbol/slot/frequency
   position per TS 38.321 (`ra_rnti.cpp`).
4. **Open USRP** and tune to the cell's UL/DL frequencies.
5. **SSB sync** — search for the gNB's SSB, decode the PBCH/MIB, and recover
   frame timing so transmissions align to the gNB's clock (`ssb_sync.cpp`).
6. **Transmit** the preamble(s) on each RO, optionally continuously.

With `--gnb-log-path` set, the tool also tails the gNB log and records which
of its preambles the gNB actually detected, closing the loop for analysis
(`gnb_detect_reader.cpp`). All transmit and detection events are written to
CSV (`log_csv.cpp`).

### Flood strategies

- **superimpose** — all N preamble indices are summed into one buffer and sent
  in a single RO. This is the high-impact mode: one occasion, N apparent UEs.
- **cycle** — a different preamble index is sent on each successive RO.

Superimposing many Zadoff-Chu sequences raises the peak-to-average power
ratio (PAPR); left unmanaged, the summed buffer clips the USRP DAC and the
waveform is destroyed. The tool manages this with per-preamble phase
optimization — Newman phases by default, or an SLM (selective mapping) search
over `--slm-candidates` random phase sets that picks the lowest-PAPR
combination — followed by peak-aware normalization before transmission
(`prach_tx.cpp`).

`--flood-n N` is shorthand for indices `0..N-1`. The underlying
`flood.num_preambles` config also accepts explicit lists and ranges, e.g.
`0,4,8` or `0-15,32`.

## Build

Requires a C++17 toolchain, UHD (tested with 4.9), yaml-cpp, and a srsRAN_4G
PHY built from source (the tool links `libsrsran_phy` for NR preamble
generation and SSB search). The InfluxDB read path uses `influxdb.hpp` from
the rt-recon-sdk.

```bash
mkdir -p build && cd build
cmake ..
make -j
```

The binary is `build/prach-inject`.

## Usage

RF transmission requires `--confirm-rf-isolated` as a safety interlock. Use
`--dry-run` to validate config and timing without transmitting.

Single preamble on every RO:

```bash
./build/prach-inject \
  --tx-gain 65 \
  --continuous \
  --confirm-rf-isolated
```

Flood — 32 superimposed preambles per occasion, continuous:

```bash
./build/prach-inject \
  --flood --flood-strategy superimpose \
  --flood-n 32 \
  --tx-gain 65 \
  --max-tx 200 \
  --continuous \
  --confirm-rf-isolated
```

Dry run (no RF, prints resolved cell + tool config):

```bash
./build/prach-inject --flood --flood-n 16 --dry-run
```

Closed-loop (record which preambles the gNB detected):

```bash
./build/prach-inject \
  --flood --flood-n 16 --tx-gain 65 --continuous \
  --gnb-log-path /tmp/gnb.log \
  --confirm-rf-isolated
```

### Key options

| Flag | Meaning |
|------|---------|
| `-c, --config PATH` | Path to `prach-inject.yaml` (default `configs/prach-inject.yaml`) |
| `--confirm-rf-isolated` | Required to transmit; guards against accidental on-air use |
| `--dry-run` | Run the full pipeline without RF transmission |
| `--tx-gain DB` | USRP TX gain (dB) |
| `--continuous` | Transmit on every RO until stopped |
| `--max-tx N` | Stop after N transmissions (0 = unlimited) |
| `--flood` | Enable multi-preamble flood mode |
| `--flood-n N` | Flood with indices `0..N-1` (1–64) |
| `--flood-strategy S` | `superimpose` or `cycle` |
| `--flood-backoff DB` | Amplitude backoff to prevent clipping |
| `--slm-candidates N` | SLM phase search count (0 = Newman phases; default 32) |
| `--freq-pos-count N` | Superimpose across N frequency positions per RO (1–16) |
| `--resync-every N` | Re-sync to SSB every N transmissions |
| `--gnb-log-path PATH` | Tail this gNB log to record detected preambles |
| `--device-args ARGS` | UHD device args (e.g. `type=b200`) |

InfluxDB connection can be overridden with `--influx-host`, `--influx-port`,
`--influx-org`, `--influx-bucket`, and `--influx-data-id`. The read token is
taken from the `INFLUX_TOKEN` environment variable.

Configuration precedence is **CLI > YAML > built-in default**; the resolved
config is printed at startup with the source of each value annotated.

## Output

- `prach-inject_msg1.csv` — per-transmission log: timestamp, channel, slot/SFN,
  RNTI, RAPID, TX gain, outcome, and flood parameters.
- `gnb_detections.csv` — when closed-loop reading is enabled, the preambles
  the gNB reported detecting, with TA, power, and SNR.

## Source layout

| File | Responsibility |
|------|----------------|
| `main.cpp` | CLI parsing, config resolution, pipeline orchestration |
| `influx_reader.cpp` | Pull cell parameters from InfluxDB (Flux queries) |
| `cell_config.cpp` | PRACH config-index → RO parameters (TS 38.211) |
| `ro.cpp` | RO timing: occasion detection and next-occasion lookup |
| `ra_rnti.cpp` | RA-RNTI computation (TS 38.321) |
| `ssb_sync.cpp` | SSB search/track, PBCH decode, frame timing recovery |
| `prach_tx.cpp` | Preamble generation, flood superposition, PAPR/SLM, TX |
| `gnb_detect_reader.cpp` | Tail gNB log, parse `PRACH: detected_preambles=[...]` |
| `log_csv.cpp` | CSV event logging |

## Safety and scope

This tool transmits in licensed 5G NR uplink bands and will disrupt a real
gNB. Operate it only inside a Faraday cage or equivalent RF-isolated
environment, against a gNB you own or are explicitly authorized to test. The
`--confirm-rf-isolated` interlock exists to make on-air transmission a
deliberate choice, not an accident.
