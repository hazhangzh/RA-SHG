# Hardware control notes: RA-SHG setup

Reference notes for the automated, polarization-resolved SHG measurement
(transmitted and reflected channels). Written before any acquisition code.

Sources:
- SR400 manual (Stanford Research Systems), the full Markdown copy.
- pylablib 1.4.5 source, `pylablib/devices/Thorlabs/kinesis.py`. The readthedocs
  site was not reachable from the build environment, so the API below was read
  straight from the source code.

---

## 1. Signal chain and what goes where

```
laser ─ HWP (KDC101 #1) ─ sample ─┬─ transmitted SHG ─ analyzer T (KDC101 #2) ─ filter ─ PMT_T ─► SR400 INPUT 1 ─► counter A
                                  └─ reflected SHG   ─ analyzer R (KDC101 #3) ─ filter ─ PMT_R ─► SR400 INPUT 2 ─► counter B
```

This wiring is **forced** by the SR400 input multiplexer (manual, `CI` command):

| Counter | Allowed inputs            |
|---------|---------------------------|
| A       | 10 MHz, INPUT 1           |
| B       | INPUT 1, INPUT 2          |
| T       | 10 MHz, INPUT 2, TRIG     |

Counter A can't count INPUT 2, so the transmitted PMT goes to **INPUT 1 → A** and
the reflected PMT to **INPUT 2 → B**. Counter T counts the internal 10 MHz clock
and sets the count period. Both channels are counted in the same window, so
they are measured simultaneously.

---

## 2. SR400 photon counter

### 2.1 Communication

| Item | RS-232 | GPIB |
|---|---|---|
| Settings | Front panel COM menu: BAUD up to 19200 (default 9600), 8 bits, no parity, **ECHO = OFF** | Address 23 (default) |
| Command terminator (host → SR400) | `\r` (CR or LF or both are accepted) | same |
| Response terminator | `\r` with echo OFF (can be changed with `SE`) | always `\r\n` |
| Connector | DB-25, configured as **DCE** → use a straight cable (not null-modem) | standard |

Gotchas:
- **Send `SW0` at startup.** The default RS-232 character wait is 6 × 3.3 ms ≈
  20 ms *per character*, so a 9-digit count takes about 200 ms to arrive. `SW0` removes the wait.
- The SR400 waits for **CTS** before it sends. If a USB-serial adapter or cable
  doesn't drive CTS, reads will hang. Use a cable that passes RTS/CTS and
  assert RTS, or enable `rtscts`.
- Multiple commands per line are separated by `;`. Read the reply to one query
  before you send the next. Never put anything else on the same line as `CL`.
- Out-of-range or unknown commands set status bit 7 and discard the rest of the line.
- Use 8N2 on the host side (the manual's example uses 2 stop bits). This is safe either way.
- Send a few bare `\r` and flush the input buffer when the program starts.

### 2.2 Counter configuration for this experiment

```
CM0          count mode A,B FOR T PRESET (A and B independent; also resets counters)
CI0,1        counter A ← INPUT 1  (transmitted PMT)
CI1,2        counter B ← INPUT 2  (reflected PMT)
CI2,0        counter T ← 10 MHz   (count period = fixed time)
CP2,1E7      T preset = 1e7 ticks of 100 ns = 1.0 s count window
NP1          N PERIODS per scan  (or NP N for N repeats per angle, see 2.3)
NE0          end-of-scan = STOP (required for reliable data readout)
DT2E-3       dwell between periods (2 ms min; only matters for NP>1)
GM0,0        A gate CW (continuously open)  ┐ for CW / high-rep-rate (MHz) lasers
GM1,0        B gate CW                      ┘
DS0,1        A discriminator slope FALL (PMT pulses are negative)
DS1,1        B discriminator slope FALL
DL0,-0.010   A disc level (V) — set from a pulse-height check, see 2.4
DL1,-0.010   B disc level (V)
SW0          no RS-232 char delay
```

**The preset uses only one significant digit.** `CP2,n` keeps the most significant
digit of n, so `CP2,15E6` becomes `1E7`. The allowed count windows are
{1–9} × 10^k × 100 ns: 0.1 s, 0.2 s, 0.5 s, 1 s, 2 s, and so on, not 1.5 s. `DT` works the same way.

### 2.3 Reading data

There are two clean ways to take one point (both use `NE0` = STOP):

**A. Start a scan and stream it (`FT`).** This is the preferred way:
```
CR                      reset counters (FT requires counters in reset)
FT                      start scan; SR400 returns A1,B1,A2,B2,...,AN,BN
read 2*NP lines         each terminated by \r
```
The read timeout must cover NP × (count window + dwell) plus a margin.

**B. Start and poll:**
```
SS                      read & clear status byte
CR;CS                   reset, start
QA1  → -1 until done    poll; returns count of point 1 of counter A
QB1                     then read B for the same point
```
Alternatively, poll `SS 2` (Scan Finished) and then dump with `ET`.

Other useful queries: `NN` gives the current scan position. `XA`/`XB` give the live
counter contents (only meaningful for long windows). `SS` is the status byte:

| bit | meaning | action |
|---|---|---|
| 1 | Data ready | — |
| 2 | Scan finished | — |
| 3 | Counter overflow (≥ 10⁹−1) | flag point, shorten window |
| 4 | Rate error (missed gate) | only relevant when gated |
| 7 | Command error | raise |

`SS` is cleared when it's read, so read it once to clear it before each acquisition.

Resetting (`CR`) or starting a new scan while an `E`/`F` dump is still streaming
aborts the dump and can hang the program. Always read all 2·NP values first.

### 2.4 Discriminator / PMT considerations (H10721-210)

- The H10721 has a current output with negative-going pulses. The SR400 inputs are
  50 Ω, linear to ±300 mV, and the **minimum detectable pulse is 10 mV**. Look at the
  PMT output into 50 Ω on a fast scope at the intended control voltage. If
  single-photon pulses are only a few mV, a preamp is needed (e.g. SR445A, ×5 per stage).
- To set the discriminator level, take a plateau curve. Scan the level with
  `DM i,1` / `DY i,step` / `NP` (with no light and with weak light) and pick the level
  in the flat region above the noise. The front-panel DISC outputs (NIM) show
  what is being counted.
- Count rate: pulse-pair resolution is 5 ns, but PMT/discriminator pile-up
  starts well before that. Keep the rates within the linear range (roughly ≲ 1–2 MHz,
  to be checked). SR400 supports two-discriminator pile-up correction (manual
  §"2 Photon Pile Up Correction"), but it uses both A and B on one input, so it
  isn't available while we count two PMTs.
- Gating (`GM i,1`, `GD`, `GW`, `TL`, `TS`) only helps with a pulsed source whose
  trigger is ≤ 1 MHz (e.g. a kHz amplifier). Then T can count TRIG for a
  fixed number of gates per window. For an ~80 MHz oscillator, use CW gates.

---

## 3. Thorlabs KDC101 rotation mounts via pylablib

### 3.1 Install / connection

```
pip install pylablib          # pulls in pyft232, pyserial, numpy, ...
```
- Windows: the Thorlabs Kinesis (or APT) software must be installed for the FTDI
  driver, but it **must be closed** while Python runs, because the port is exclusive.
- Linux: the device appears through FTDI (VID:PID 0403:faf0). This works through the
  ft232 backend (libftdi) or `/dev/ttyUSB*` with the serial backend.
- The link runs at 115200 baud with RTS/CTS, and pylablib configures it itself.

```python
from pylablib.devices import Thorlabs
Thorlabs.list_kinesis_devices()          # -> [("27xxxxxx", "Kinesis K-Cube  DC Driver"), ...]
m = Thorlabs.KinesisMotor("27xxxxxx", scale="stage")   # or scale="PRM1-Z8"
m.get_device_info(); m.get_stage(); m.get_scale_units()  # -> "PRM1-Z8", "deg"
```

**Set `scale`.** The default is `scale="step"`, which means raw encoder counts.
- `scale="stage"` lets the KDC101 report its stage ID and pylablib maps it:
  `PRM1-Z8` gives **1919.6418578623391 counts/deg**, and `CR1-Z7` is also known.
- `scale="PRM1-Z8"` sets the stage explicitly. This is the safest choice if the stage ID in the
  controller was never configured.
- With the stage set, positions are in deg, velocities in deg/s and accelerations in deg/s².
  For the KDC101, pylablib computes the velocity/acceleration scale from the 2048/6 MHz time base.

### 3.2 API (KinesisMotor, single-channel KDC101)

| Call | Behaviour |
|---|---|
| `home(sync=True, force=False, timeout=None)` | Homes the stage. **Skipped if already homed** unless `force=True`. `sync` blocks until done. |
| `is_homed()`, `is_homing()`, `wait_for_home(timeout=None)` | |
| `move_to(position)` | Absolute move, **non-blocking** |
| `move_by(distance)` | Relative move, non-blocking |
| `wait_move(timeout=None)` | Blocks until no `moving_*`/`jogging_*`/`active` status bit is set. Raises `ThorlabsTimeoutError` on timeout. |
| `is_moving()` | |
| `get_position()` | Reads the position from the encoder (deg) |
| `set_position_reference(position=0)` | Redefines the current position (no motion) |
| `stop(immediate=False, sync=True)` | Profiled or abrupt stop |
| `get_velocity_parameters()` / `setup_velocity(min_velocity=None, acceleration=None, max_velocity=None)` | min_velocity always 0 |
| `get_homing_parameters()` / `setup_homing(home_direction=None, limit_switch=None, velocity=None, offset_distance=None)` | |
| `get_gen_move_parameters()` / `setup_gen_move(backlash_distance=None)` | Controller-side backlash correction |
| `get_status()` | List of strings, e.g. `"homed"`, `"moving_fw"`, `"motion_error"`, `"enabled"` |
| `close()` / `with KinesisMotor(...) as m:` | |
| `scale=False` keyword on most calls | Uses raw counts instead of physical units |

Practical notes:
- **No angle wrap-around.** The position is a signed 32-bit count, so `move_to(370)`
  goes to 370° absolute, not to 10°. Plan target angles as an unwrapped, monotonic sequence,
  or reduce them modulo 360 and accept the long way round.
- Moves are non-blocking. Start all three motors and then `wait_move()` on each,
  so they move in parallel.
- After `wait_move()`, read back `get_position()` and **log the actual angle**
  (encoder resolution ≈ 0.0005°), not just the requested one.
- The controller applies backlash correction on moves in the reverse direction. For the best
  repeatability, also approach every point from the same direction.
- Homing a PRM1-Z8 can take tens of seconds. Do it once at startup (`force=False`).

---

## 4. Polarization geometry for the scan

- An HWP at physical angle θ rotates linear polarization by twice its angle:
  φ_in = φ₀ + 2·(θ_HWP − θ_HWP,0). For a polarization step Δφ, the HWP moves Δφ/2.
- Each analyzer co-rotates with the incident polarization:
  θ_an = θ_an,0 + s·(φ_in − φ₀) + δ, where δ = 0° for the parallel channel and 90° for the perpendicular.
  The analyzers therefore move twice as far per point as the HWP.
- `s = ±1` is a per-mount sign. It depends on which way the mount faces the beam, and
  for the reflected arm the handedness of the angle reverses on reflection. It has to
  be set by calibration, not assumed.
- Calibration constants to determine once (and store in a config file):
  θ_HWP,0, θ_T,0, θ_R,0, s_T, s_R. For example, find each analyzer's extinction
  against a reference polarizer, and the HWP zero against the analyzer.

Per-point measurement loop (outline):
1. Compute the three targets from φ. `move_to` all three, then `wait_move` all three.
2. Read back the three positions.
3. Acquire: `CR`, `FT`, read A₁B₁…A_NB_N. Check the `SS` overflow/error bits.
4. Store φ, the three actual positions, A and B (mean, std, raw), the count window, and a timestamp.

---

## 5. Open questions (needed before writing code)

1. Rotation stage model on each KDC101 (PRM1-Z8?), and the three serial numbers.
2. SR400 link: RS-232 (through which USB-serial adapter?) or GPIB (which adapter)?
3. Laser repetition rate: CW gates, or gated counting with a trigger?
4. Analyzer configurations per scan: parallel only, perpendicular only, or both
   (two passes, or ∥ and ⊥ at each angle)?
5. Acquisition PC OS (Windows + Kinesis drivers assumed).
6. Desired angle range/step and count window per point.
