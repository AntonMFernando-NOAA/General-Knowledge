# How ATM and OCN Outputs Work in GFSv17

## The 6-Hour Cycling Clock

The system runs 4 cycles per day at 00, 06, 12, and 18 UTC. Each cycle:

1. Runs data assimilation (analysis) to get the best current state
2. Launches a forecast from that state

GDAS runs short forecasts (9h) for cycling. GFS runs long forecasts (384h/16 days) for products.

---

## ATM (FV3) — Atmosphere

### Timing (example: GFS 18z cycle)

- Model starts at **12z** (cycle - 6h, for IAU)
- IAU applies 3 increments at model hours 3, 6, 9 (= 15z, 18z, 21z)
- Each increment has a 6h influence window (centered, overlapping)
- Free forecast (no corrections) begins at ~00z (model hour 12)
- "Forecast hour 0" from the workflow = 18z (cycle time)

### Output type

| | GDAS | GFS |
|---|---|---|
| Type | Instantaneous snapshots | Instantaneous snapshots |
| Frequency | Every 1-3h | Every 1-3h (hourly for HF, 3-hourly after) |
| What `f006` means | Snapshot at exactly fhr6 | Snapshot at exactly fhr6 |

### How ATM output works

- FV3 writes output files directly as it runs
- FV3 writes its own sentinel: `log.atm.fHHH`
- The sentinel means "I finished writing the data file for this hour"
- Forecast manager polls for `log.atm.fHHH`, then copies the data file to COM

### File naming (GFS 18z)

```
Data:      atmf006.nc              (atmosphere at fhr006 = 00z)
Sentinel:  log.atm.f006            (written by FV3 when file is complete)
COM dest:  gfs.t18z.atmf006.nc
COM log:   gfs.t18z.log.atm.atmf.f006.txt
```

### ATM IAU — how the analysis feeds in

```
GDAS 12z produces backgrounds at 09z, 12z, 15z
→ 18z ATM analysis uses those + observations
→ produces 3 increments (valid at 15z, 18z, 21z)
→ GFS 18z ATM applies them during model hours 0-12

After model hour 12 (= 00z): pure free forecast
```

---

## OCN (MOM6) — Ocean

### Timing (example: GFS 18z cycle)

- MOM6 starts at **21z** (cycle + 3h)
- ODA_INCUPD applies a single increment over 6 hours (21z–03z)
- Free forecast begins at 03z
- Runs for 384 hours total (16 days)

### Output type

| | GDAS | GFS |
|---|---|---|
| Type | Instantaneous snapshots | 6-hour period averages |
| Frequency | Every 1h | Every 6h |
| What `f006` means | Snapshot at exactly that hour | Average over hours 0-6 |
| Why averages? | DA needs instants for obs comparison | Ocean is noisy at instants; averages are physically meaningful |

### How OCN output works

- MOM6's **diag_manager** writes the `.nc` data file when an averaging period completes
- MOM6's **outputlog** feature (alarm-based) monitors the file and writes the sentinel
- The alarm rings every 6h (GFS) or 1h (GDAS)
- When the alarm rings: construct filename → check if file is complete → write sentinel

### The alarm and sentinel mechanism (GFS, 6-hourly)

```
Alarm rings → "which file should I look for?" → constructs filename
→ polls: "is it complete?" (checks file size stopped growing)
→ YES → writes sentinel: YYYYMMDD.HHMMSS.mom6.06h
→ forecast manager sees sentinel → copies data file to COM
```

For the last file: special `lstop` sentinel (`mom6.lstop.06h`) written at model stop.

### File naming (GFS 18z)

```
Data:      ocn_2026_08_08_00_00.nc      (6h avg, midpoint at 00z = fhr006)
Sentinel:  20260808.030000.mom6.06h     (valid time in sentinel name)
COM dest:  gfs.t18z.6hr_avg.f006.nc
COM log:   gfs.t18z.6hr_avg.log.f006.txt
```

### OCN IAU (ODA_INCUPD) — how the analysis feeds in

```
GDAS 12z OCN produces hourly backgrounds (16z, 17z, 18z, ..., 00z)
→ 18z marine analysis (JEDI/SOCA) uses 18z background + ocean obs
→ produces 1 increment: gfs.t18z.mom6_increment.i006.nc
→ GFS 18z MOM6 applies it: 1/6 each hour for 6 hours (21z-03z)

After 03z: pure free forecast
```

---

## OCN (MOM6) — GDAS Cycling Example (12z cycle)

### What happens

```
MOM6 starts:    15z (12z + 3h)
Runs:           9 model hours (15z → 00z)
ODA:            Single increment applied over 6h (15z–21z)
Free forecast:  21z–00z (3 hours)
Output:         Hourly instantaneous snapshots
```

### Hourly output (all copied to COM)

```
16z, 17z, 18z, 19z, 20z, 21z, 22z, 23z, 00z  (9 files)
```

### Sentinel naming (GDAS, hourly)

```
Data:      ocn_2026_08_07_18_00.nc
Sentinel:  20260807.180000.mom6.01h    (.01h = hourly alarm)
COM dest:  gdas.t12z.inst.f006.nc
```

### What feeds the next cycle

```
OCN backgrounds (esp. 18z snapshot) → 18z marine analysis
Restart at 21z → GFS 18z MOM6 start / GDAS 18z MOM6 start
```

---

## Side-by-Side Comparison

|  | ATM (FV3) | OCN (MOM6) |
|---|---|---|
| Start time (18z cycle) | 12z (cycle - 6h) | 21z (cycle + 3h) |
| IAU method | 4D-IAU: 3 increments at hrs 3,6,9 | ODA_INCUPD: 1 increment over 6h |
| Free forecast from | 00z (model hr 12) | 03z (model hr 6) |
| GFS output | Instantaneous every 1-3h | 6h averages every 6h |
| GDAS output | Instantaneous every 1-3h | Instantaneous every 1h |
| Sentinel | `log.atm.fHHH` (written by FV3 directly) | `YYYYMMDD.HHMMSS.mom6.HHh` (written by outputlog alarm) |
| Who writes sentinel | FV3 itself | MOM6's outputlog (alarm-triggered) |
| Analysis system | GSI / JEDI-Atm | JEDI/SOCA (marine) |
| Increment file | `atminc.fHH.nc` (3 files) | `mom6_increment.i006.nc` (1 file) |
| Restart frequency (GDAS) | Every 3h | Every 3h |
| Restart frequency (GFS) | Every 48h | Every 48h |

---

## The Full Chain: GDAS 12z → GFS 18z

```
GDAS 12z ATM (06z–21z):
  Produces ATM backgrounds → feed 18z ATM analysis

GDAS 12z OCN (15z–00z):
  Produces OCN backgrounds → feed 18z marine analysis
  Writes restart at 21z   → GFS 18z MOM6 loads this

18z Analyses (ATM + marine, run in parallel):
  ATM analysis → 3 increments for GFS 18z ATM
  Marine analysis → 1 increment for GFS 18z OCN

GFS 18z (12z–Day 16):
  ATM: starts 12z, IAU through 00z, free forecast 384h
  OCN: starts 21z, ODA through 03z, free forecast 384h
  Both produce output throughout → forecast manager copies to COM
```

---

## Restarts

Restarts are the initial conditions for warm starts. Each cycle picks up where the last left off.

- Written by CMEPS (coupler) at configured intervals
- GDAS: every 3h (so next cycle can start at the right time within the IAU window)
- GFS: every 48h (crash recovery only)
- File: `YYYYMMDD.HHMMSS.MOM.res.nc` (+ tiles for 1/4° resolution)
- The ocean state is continuous — never restarted from scratch during normal operations

---

## The Coupler (CMEPS)

CMEPS sits between ATM, OCN, and ICE:

- Passes fields between components every coupling timestep (~1h)
- Regrids between different grids (cubed-sphere ATM, tripolar OCN)
- Controls the run sequence (which component advances when)
- Triggers restart writes for all components
- Built on ESMF/NUOPC framework (same framework that provides clocks and alarms)
