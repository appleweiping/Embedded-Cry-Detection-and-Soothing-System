# Embedded Cry Detection and Soothing System

A rocking-cradle controller for a crying infant, built as a TU/e team project on four PYNQ-Z2 boards:
one board listens for crying, one measures the baby's heart rate with a photodiode, one drives the
rocking motor, and one decides how hard and how fast to rock.

The board programs need the course's `libpynq` library and the hardware, so they cannot run on a
laptop. What does run anywhere is `sim/sim.c`: a simplified, earlier version of the decision
board's search, run against a made-up "stress" model of the baby. It is not the board code; the
differences are listed under [The simulator](#the-simulator).

```text
$ gcc -Wall -O2 -o sim/sim sim/sim.c -lm && ./sim/sim 1792644429
seed=1792644429
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

The simulated baby is random. With no argument the simulator seeds it from the clock, so every run
is different; it prints the seed first, and passing that seed repeats the run. `rand()` differs
between C libraries, so the seed above reproduces this exact output only with glibc (Linux); MSYS2
gcc on Windows gives a different run for the same seed.

Over seeds 1 to 1000 on Ubuntu 24.04 (gcc 13.3) the simulated controller reached the gentlest
setting in every run, after 8 to 12 controller steps, i.e. 108 to 168 simulated seconds. Seeds 1 to
300 with MSYS2 gcc 14.2 on Windows gave the same range. To count them yourself:

```bash
for s in $(seq 1000); do ./sim/sim $s | awk '/Controller Step/{n++} /finished/{print n, "steps,", $5, "s"}'; done | sort -n | uniq -c
```

All this shows is that the simulator's search finishes against the simulator's baby. It is not a
result for the board controller: the simulator judges each step with information the board does
not have, and it has no panic stop (see [The simulator](#the-simulator)). I have no measurements
from the boards or from a real baby to report.

## How it works

Four boards sit on a ring of UART links. Every frame is `[DST][SRC][LEN][PAYLOAD...]`; a board
handles frames addressed to it and forwards the rest.

| Addr | Folder | Board's job |
|------|--------|-------------|
| 0 | `decision/` | Master. Pings the other boards at start-up, then once per controller step asks the heart-rate board for BPM, runs the controller and sends a motor command. Shows its state on the PYNQ's LCD. |
| 1 | `heartbeat/` | Infant heart rate. A photodiode and op-amp strapped to the baby's arm feed ADC0; the code detects beats with a peak/trough threshold and reports BPM averaged over the last 10 beat intervals. |
| 2 | `crying/` | Cry level. Samples the analog audio signal on ADC0 at 200 Hz, takes the peak-to-peak amplitude over a 200 ms window and maps it to 0-100 % using a calibration done at start-up (3 s quiet, then 5 s of loud playback). |
| 3 | `motor/` | Drives the cradle with two 1 kHz PWM outputs, one for rocking amplitude and one for rocking frequency. |

**The controller.** Rocking settings form a 5x5 grid: amplitude A1-A5 by frequency F1-F5. The
master starts at the strongest setting (A5 F5). Each step it tries a gentler cell, first lower
frequency ("LEFT"), then lower amplitude ("UP"). If the baby got calmer, the new cell becomes the
anchor; if not, it backs off to the previous anchor. "Calmer" means a heart-rate drop of at least
10 BPM, except below 150 BPM with a cry level between 15 and 52 %, where any drop in the cry level
counts instead. It waits between steps because heart rate reacts slowly (`HEARTBEAT_DELAY`,
`CRYING_DELAY` and `CONVERGENCE_DELAY` in `decision/main.c`) and stops when it reaches A1 F1. If
the heart rate jumps by 30 BPM or more between steps it enters a panic mode that parks the cradle
at A1 F1 and stops searching.

In the normal mode the main loop only reads the heart rate: it reads the crying board for up to a
second after the pings, then passes a cry level of 0 to every controller step, so the
cry-level rule above is never used there. It also only runs the controller if the motor board
answered the start-up ping.

**On the board.** The decision board draws a status screen on the PYNQ's LCD: which boards
answered the ping, the latest BPM and cry level, whether heart rate or crying is driving the
decisions, the current grid cell and motor command, whether panic mode is on, and the time since
the search started (marked CALM once it reaches A1 F1), with the latest controller log lines
underneath. In the normal mode button B3 restarts the program. Two switches pick a demo mode at
start-up:

- **Switch 0: manual vitals.** The controller runs on numbers you set with the buttons instead of
  the sensors: B0/B1 lower/raise the BPM by 10 and B2/B3 lower/raise the cry level by 10, starting
  from 220 BPM and 100 %. This is the way to try the controller on one board without sensors, and
  the only mode that reaches the cry-level rule. The first time BPM drops below 150 the program
  sets the cry level to 52, just outside that rule's range; one B2 step brings it inside. The
  buttons are read once per controller step (every 4 to 14 s), so hold a button until its value
  changes on the screen. Turning the switch off ends the program.
- **Switch 1: link test.** Pings the other boards and shows their live readings; B0 and B1 send two
  fixed motor commands. Turning the switch off continues into the normal mode.

### The simulator

`sim/sim.c` is a simplified, earlier version of the same LEFT/UP search, written so the search
could be tried without hardware. It differs from `decision/main.c` in ways that matter:

- It judges each step by heart rate only; it has no cry-level rule.
- Its `heartbeat_improved()` counts a step as better whenever the simulated baby's hidden stress
  level went down (`curK < K[prevA][prevF]`), which the board cannot know, and also whenever BPM is
  70 or lower. The board only counts a drop of at least 10 BPM. When a LEFT move leaves the hidden
  level unchanged, the simulator also switches to UP straight away.
- "PANIC JUMP" and "PANIC BLOCK" in its output are part of the baby model: a move between two
  settings whose stress ranges don't overlap can send the simulated baby to the highest stress level.
  The controller doesn't react to them and keeps searching (527 of the 1000 runs above contain one).
  The board's 30 BPM panic stop is not in the simulator.

## Building for the boards

Each module folder is a libpynq application. The Makefiles include `../shared.mk` and `../end.mk`,
so they only build inside the course library (`libpynq-5EWC0-2023`, not included here). The
binaries that used to be committed here were built against v0.2.6 of it:

```bash
# on the PYNQ board, with the course library unpacked in ~/libpynq-5EWC0-2023-v0.2.6
cp -r crying decision heartbeat motor ~/libpynq-5EWC0-2023-v0.2.6/applications/
cd ~/libpynq-5EWC0-2023-v0.2.6/applications/decision && make && ./main
```

Run `make` the same way in the other three folders, one program per board, and connect the boards'
UART pins in a ring in address order.

## Repository layout

```text
crying/  decision/  heartbeat/  motor/   the four board programs (main.c + Makefile)
sim/sim.c                                simplified desktop simulator of the controller
Parts/                                   CAD for the photodiode strap: v2 as STEP, v3/v4 SolidWorks only
alt-versions/                            other versions of three modules, see below
```

`alt-versions/` holds other versions of three modules that used to sit in the module folders next
to `main.c` (old file names in brackets):

| File | Was |
|------|-----|
| `alt-versions/crying/crying_alt.c` | `crying/CRYING final` |
| `alt-versions/crying/crying_readable.c` | `crying/readable crying.c` |
| `alt-versions/heartbeat/heartbeat_readable.c` | `heartbeat/readable heartbeat.c` |
| `alt-versions/motor/motor_pwm_alt.c` | `motor/PWM Final` |

Despite the old "final" names, the file `make` builds in each module is `main.c`. These versions
differ from it, and the compiled binaries that used to be committed were built from code closer to
these files than to `main.c`, so I can't tell which version ran in the final demo. I moved them out
of the module folders because `make` compiles every `*.c` file there and failed on the second
`main()`. There was also a `readable heartbeat` without the `.c`; it was the same as
`heartbeat_readable.c` minus the Ctrl+C handler, so I deleted it (it is still in the git history,
and it is the closest match to the old heartbeat binary).

CI ([`.github/workflows/sim.yml`](.github/workflows/sim.yml)) builds and runs the simulator on
every push.

## Team

This was a TU/e group project, with the work split across the four modules. The git history here
only shows my upload of the finished code, not who wrote which part, so I don't claim any single
module as mine.

## License

MIT, see [LICENSE](LICENSE). `libpynq` is the course's library and is not part of this repository.
