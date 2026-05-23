# Embedded Cry Detection and Soothing System

Distributed embedded system on PYNQ hardware that detects infant crying via ADC audio sampling and autonomously triggers soothing responses. Four nodes communicate over a ring UART bus.

---

## System Architecture

Four PYNQ nodes connected in a ring topology via UART:

| Address | Module | File | Role |
|---------|--------|------|------|
| 0 | Master / Decision | `decision/main.c` | Coordinates all nodes, implements soothing state machine |
| 1 | Heartbeat | `heartbeat/` | Monitors system vitals, reports to master |
| 2 | Cry Detection | `crying/main.c` | ADC sampling at 200 Hz, peak-to-peak windowing, calibration |
| 3 | Motor | `motor/main.c` | PWM-driven rocking mechanism, responds to master commands |

**Ring UART frame format:** `[DST][SRC][LEN][PAYLOAD...]`

Each node forwards frames not addressed to it, creating a ring bus.

---

## Cry Detection

The detection module samples audio via ADC at 200 Hz and uses a sliding peak-to-peak window (200 ms, configurable) to measure signal amplitude.

**Calibration sequence:**
1. 3 s quiet baseline capture
2. 5 s loud playback for maximum reference
3. Threshold derived from calibration data

**Key constants** (in `crying/main.c`):

```c
#define TIME_BETWEEN_SAMPLES_MS  5     // 200 Hz
#define P2P_WINDOW_MS            200   // peak-to-peak window
#define CAL_BASELINE_MS          3000  // quiet calibration
#define CAL_MAX_MS               5000  // loud calibration
```

---

## Decision Logic

The master node (`decision/main.c`) polls heartbeat and cry modules every 100 ms and drives the soothing state machine:

```c
#define HEARTBEAT_DELAY   14000   // ~10 s heartbeat period
#define CRYING_DELAY       4000   // ~2 s cry response delay
#define CONVERGENCE_DELAY  4000   // convergence timeout
```

On cry detection, the master sends a motor command to start rocking. On convergence (crying stops), it sends a stop command.

---

## Building

Each module has its own `Makefile` targeting the PYNQ board:

```bash
cd crying && make
cd decision && make
cd motor && make
cd heartbeat && make
```

Requires `libpynq` (included in the course library at `libpynq-5EID0-2023-v0.3.0/`).

---

## Project Layout

```
crying/       Cry detection — ADC sampling, calibration, peak-to-peak detection
decision/     Master node — state machine, UART coordination
motor/        Motor control — PWM output, rocking actuation
heartbeat/    Heartbeat monitor — system vitals
sim/          Simulator for offline testing (sim.c)
Parts/        Hardware design files (CAD)
```

---

## Result

Automated intervention reduced crying duration to under 2 minutes in testing.

---

## License

MIT
