# Embedded Cry Detection and Soothing System

A rocking-cradle controller for a crying infant, built as a TU/e team project on four PYNQ-Z2 boards:
one board listens for crying, one measures the baby's heart rate with a photodiode, one drives the
rocking motor, and one decides how hard and how fast to rock.

The board programs need the course's `libpynq` library and the hardware, so they cannot run on a
laptop. What does run anywhere is `sim/sim.c`, a desktop simulator of the decision logic against a
made-up "stress" model of the baby:

```text
$ gcc -Wall -O2 -o sim/sim sim/sim.c -lm && ./sim/sim
K1: Sopt= 11.0  range=[  0.0,  23.0]
...
K9: Sopt= 75.0  range=[ 63.0,  87.0]

K1 K3 K4 K6 K8
K2 K3 K4 K6 K8
K5 K5 K5 K6 K8
K7 K7 K7 K7 K8
K9 K9 K9 K9 K9
[init] pos=A5 F5  K9  S=75.0  (band 63.0-87.0  Sopt=75.0) @t=0.00

[ALGORITHM] Controller Step 1
[SENSE] S_tau=75.0  BPM=195  CRY=100  pos=A5 F5 K9 @t=10.00
[ALGORITHM] initial/pick -> try LEFT from A5 F5
...
[ALGORITHM] Controller Step 11
[SENSE] S_tau=22.0  BPM=100  CRY=30  pos=A2 F1 K2 @t=150.00
[ALGORITHM] last move (dir=1) IMPROVED -> new anchor at A2 F1
[ALGORITHM] improved -> next try UP from A2 F1

[SYSTEM] MOVE request: A2 F1  K2 ---> A1 F1  K1
[SYSTEM] inside-band[[SYSTEM]converged] pos=A1 F1  K1  S=11.0  (band 0.0-23.0  Sopt=11.0) @t=154.00
[ALGORITHM] rest reached
finished at t = 154.000 s.
```

The simulator randomises its model on every run. In 15 runs on Ubuntu 24.04 (gcc 13.3) it reached
the gentlest setting after 9 to 12 controller steps, i.e. 122 to 164 simulated seconds. These are
simulator numbers only; I have no measurements from a real baby to report.

## How it works

Four boards sit on a ring of UART links. Every frame is `[DST][SRC][LEN][PAYLOAD...]`; a board
handles frames addressed to it and forwards the rest.

| Addr | Folder | Board's job |
|------|--------|-------------|
| 0 | `decision/` | Master. Pings the other boards at start-up, asks for heart rate and crying level every 100 ms, runs the controller and sends motor commands. |
| 1 | `heartbeat/` | Infant heart rate. A photodiode and op-amp strapped to the baby's arm feed ADC0; the code detects beats with a peak/trough threshold and reports BPM averaged over the last 10 beat intervals. |
| 2 | `crying/` | Cry level. Samples the analog audio signal on ADC0 at 200 Hz, takes the peak-to-peak amplitude over a 200 ms window and maps it to 0-100 % using a calibration done at start-up (3 s quiet, then 5 s of loud playback). |
| 3 | `motor/` | Drives the cradle with two 1 kHz PWM outputs, one for rocking amplitude and one for rocking frequency. |

**The controller.** Rocking settings form a 5x5 grid: amplitude A1-A5 by frequency F1-F5. The
master starts at the strongest setting (A5 F5). Each step it tries a gentler cell, first lower
frequency ("LEFT"), then lower amplitude ("UP"). If the baby got calmer, the new cell becomes the
anchor; if not, it backs off to the previous anchor. "Calmer" means a heart-rate drop of at least
10 BPM, except below 150 BPM with a cry level between 15 and 52 %, where it goes by the cry level
instead. It waits between steps because heart rate reacts slowly (`HEARTBEAT_DELAY`,
`CRYING_DELAY` and `CONVERGENCE_DELAY` in `decision/main.c`) and stops when it reaches A1 F1. If
the heart rate jumps by 30 BPM or more between steps it enters a panic mode that parks the cradle
at A1 F1 and stops searching. `sim/sim.c` runs the same search against a simulated baby so the
logic could be tested without hardware.

## Building for the boards

Each module folder is a libpynq application. The Makefiles include `../shared.mk` and `../end.mk`,
so they only build inside the course library (`libpynq-5EID0-2023`, not included here):

```bash
# on the PYNQ board, with the course library unpacked in ~/libpynq-5EID0-2023-v0.3.0
cp -r crying decision heartbeat motor ~/libpynq-5EID0-2023-v0.3.0/applications/
cd ~/libpynq-5EID0-2023-v0.3.0/applications/decision && make && ./main
```

Run `make` the same way in the other three folders, one program per board, and connect the boards'
UART pins in a ring in address order.

## Repository layout

```text
crying/  decision/  heartbeat/  motor/   the four board programs (main.c + Makefile)
sim/sim.c                                desktop simulator of the controller
Parts/                                   CAD for the photodiode strap (v4 fits a fitness-band strap)
alt-versions/                            other versions of three modules, see below
```

`alt-versions/` holds files that were in the module folders next to `main.c`: `CRYING final`,
`readable crying.c`, `readable heartbeat`, `readable heartbeat.c` and `PWM Final`. They differ from
`main.c`, and the compiled binaries that used to be committed were built from code closer to these
files than to `main.c`, so I can't tell which version ran in the final demo. I moved them out of
the module folders because `make` compiles every `*.c` file there and failed on the second `main()`.

CI ([`.github/workflows/sim.yml`](.github/workflows/sim.yml)) builds and runs the simulator on
every push.

## Team

This was a TU/e group project, with the work split across the four modules. The git history here
only shows my upload of the finished code, not who wrote which part, so I don't claim any single
module as mine.

## License

MIT, see [LICENSE](LICENSE). `libpynq` is the course's library and is not part of this repository.
