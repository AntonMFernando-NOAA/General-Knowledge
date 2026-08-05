# Chapter 3.1 — How the Forecast Manager Works

> *The model writes hundreds of output files. Something has to copy each one to
> the official storage the moment it's finished — no sooner, no later, and
> without ever publishing a half-written file. That something is the forecast
> manager. This chapter explains what it is, how it knows a file is "done," and
> how it handles the three components (atmosphere, ocean, ice) that behave
> differently.*

---

## Section 1 — The problem it solves

A coupled forecast (FV3 atmosphere + MOM6 ocean + CICE ice + WW3 waves) writes
its output to a fast scratch directory (`DATAoutput`) while it runs. Downstream
jobs — post-processing, product generation, archiving — read from the official
**COM** directory instead. So each output file has to be copied from scratch to
COM.

The tricky part is *timing*:

- Copy **too early** and you publish a half-written file. Downstream jobs then
  read garbage.
- Copy **too late** (e.g., wait until the whole forecast finishes) and you delay
  every downstream job unnecessarily.

The forecast manager copies each file **as soon as it is safely complete**, in
parallel with the still-running model. It's the piece that lets products flow
out hour-by-hour instead of all at the end.

```
model (running)                forecast manager                downstream
┌───────────────┐   writes    ┌──────────────────┐   copies   ┌───────────┐
│ FV3 / MOM6 /  │────────────▶│ watch for "done", │──────────▶│ post,     │
│ CICE / WW3    │  to scratch │ copy to COM       │  to COM   │ products, │
└───────────────┘             └──────────────────┘            │ archive   │
                                                              └───────────┘
```

---

## Section 2 — The key idea: sentinels

How does the manager know a file is finished and not still being written?

The model drops a tiny companion file — a **sentinel** — next to each output
file once that output is fully flushed to disk. Think of it as a "✓ done" sticky
note. The manager watches for the sentinel, not the data file itself, because a
data file *appears* on disk long before it's finished writing.

For the atmosphere, the sentinel for forecast hour 6 is a file named
`log.atm.f006`. When it appears, `atmf006.nc` and `sfcf006.nc` are safe to copy.

The whole system is built around this **sentinel contract**: data is only
published after its sentinel proves it's complete, and the manager writes its own
"done" marker into COM so downstream jobs have the same guarantee.

---

## Section 3 — The moving parts

The manager isn't one script. It's a small team:

```
JGLOBAL_FORECAST_MANAGER            (the batch job)
│  runs
▼
exglobal_forecast_manager.sh        (builds the command list, launches MPMD)
│  runs via run_mpmd.sh (srun --multi-prog / mpiexec)
▼
┌──────────────┬──────────────┬──────────────┬─────────────┐
│ manager rank │ manager rank │ manager rank │ barrier rank│
│   (parallel) │   (atm)      │   (ocn)      │   (ice)      │   (atm)     │
└──────────────┴──────────────┴──────────────┴─────────────┘
each runs forecast_manager.sh <component> <table>
```

- **`exglobal_forecast_manager.sh`** builds a *command file* — one line per rank —
  and hands it to `run_mpmd.sh`, which launches them all in parallel (via `srun`
  on Hera/Orion-type machines, `mpiexec` on WCOSS2).
- **`forecast_manager.sh`** is the actual worker. One copy runs per component (and
  per product type for the atmosphere). It reads a **product table** and copies
  files to COM.
- **`forecast_atm_barrier.sh`** is a special atmosphere-only rank that fuses the
  per-product "done" notes into one combined per-hour sentinel (more in Section 7).

### The product table

Every worker reads a plain-text table. Each row has **four columns**:

```
local_data_file   local_log(sentinel)   com_dest_file   com_dest_log
```

| Column | Meaning |
|---|---|
| `local_data` | the file the model wrote in scratch |
| `local_log`  | the sentinel to wait on ("is it done?") |
| `com_data`   | where the data file should land in COM |
| `com_log`    | the "done" marker to write in COM after copying |

The tables are built earlier, during forecast setup, by `forecast_postdet.sh`.

---

## Section 4 — The life of one row

Each worker loops, polling the table until every row is done. For a single row it
does this:

```
1. Already marked done?            → skip
2. Is the sentinel present?        → if not, either wait, or (ocn/ice) use a fallback
3. Is the COM "done" marker there? → yes: it was done on an earlier run; mark & skip  (RERUN safety)
4. Copy the DATA files first       → with a "not still being written" check; defer if not ready
5. Write the COM "done" marker LAST
6. Mark all rows sharing this sentinel as done
```

Two rules make this safe:

- **Data first, marker last.** The COM "done" marker is only written *after* the
  data is fully copied. A downstream job that keys off the marker can never see a
  marker without its data behind it.
- **Defer, don't fail.** If a data file isn't visible yet (filesystem lag), or two
  size snapshots taken a few seconds apart disagree (still being written), the
  worker *defers* the row and retries on the next poll — it doesn't crash.

---

## Section 5 — The three ways a "done" marker gets created

This is the heart of the manager. When it's time to write the COM marker, exactly
one of three branches runs:

```
if   history-done fallback   → write a SYNTHETIC note ("model sentinel unavailable")
elif data-file trigger       → write a SYNTHETIC note ("created from data-file trigger")
else                         → COPY the model's real sentinel into COM
```

### Scenario 1 — Normal (the common case)

The model wrote a real sentinel and it exists. The worker copies the data, then
**copies the model's sentinel** into COM as the official marker. Used by the
atmosphere for every hour.

### Scenario 2 — History-done fallback (ocean & ice only)

Sometimes the per-file sentinel legitimately never appears:

- **Ocean (MOM6):** the period log is written at the *start of the next* averaging
  period, so the final window's log never shows up.
- **Ice (CICE):** the sentinel can be missing due to filesystem lag or a
  forecast-hour labeling offset.

So for `ocn`/`ice` **only**, if (a) the shared "all history finished" flag exists
and (b) the data file is present, the worker copies anyway, runs a **size sanity
check** (Section 6), and **writes its own synthetic note** recording that the
model's sentinel was missing.

This branch is gated by an explicit component-name check — it will not fire for
the atmosphere or waves.

### Scenario 3 — Data-file trigger (name-based, any component)

Some rows don't wait on a text sentinel at all. Their "wait-on" column is a
**stand-in signal**, recognized by its name:

- `*.nc` — a data file whose appearance implies an earlier file is done.
- `fcst_finalized_seg*` — a flag meaning the whole forecast job finished.
- `fcst_history_done_seg*` — a flag meaning all history was written (currently
  recognized but not used by any row).

Here the worker must **not** copy the trigger file (that would dump a big NetCDF
into COM under a log name). Instead it **writes a small synthetic note**.

Unlike Scenario 2, this is decided by the *name*, not the component. In the
current tables it fires on exactly two rows: the atmosphere's `fcst_finalized`
marker and the ice **initial-condition (IC)** row.

| | Scenario 1 Normal | Scenario 2 History-done | Scenario 3 Data-file trigger |
|---|---|---|---|
| Wait-on file | real text sentinel | (sentinel missing) | a `.nc`/flag stand-in |
| "Ready" decided by | sentinel exists | history-done flag + data present | stand-in file exists |
| Size check? | no | **yes** | no |
| COM marker | **copy** model's sentinel | **write** synthetic note | **write** synthetic note |
| Restricted to | — | `ocn`/`ice` only | any component (by name) |

---

## Section 6 — The size check (and its blind spot)

In Scenario 2 there's no trusted sentinel, so after copying, the worker sanity-
checks the file size. It looks for an **already-copied file of the same type at a
different hour** to use as a reference:

- It strips the forecast-hour token from the name (`inst.f006.nc` → `inst.fNNN.nc`)
  so it only compares like with like.
- If the new file is smaller than the reference by more than a threshold
  (currently 1 MB), it's treated as a likely partial/corrupt file.

The blind spot worth knowing:

- **The first hour of each type has no reference**, so its size check is
  **skipped** (logged as "no reference available yet") and the file is copied
  unvalidated. It then *becomes* the reference for later hours.
- Consequence: if the very first file is the truncated one, it slips through — and
  worse, a too-small first file becomes a bad baseline that can make later *good*
  files look wrong.

A purely comparative check can't protect the first element. An absolute/expected-
size floor (or retry-then-fail) would close that gap.

---

## Section 7 — How the three components differ

The same worker code runs for every component, but the tables are shaped
differently, so each behaves in its own way.

### Atmosphere (ATM) — groups + a barrier

ATM is the only component that truly copies files in **groups**, and it runs as
**four parallel product ranks**: `atmf`, `sfcf`, `grib`, `flux`.

- **A group = rows sharing one sentinel** `log.atm.fHHH` inside a rank's table.
  Example: for hour 6 the `atmf` rank may hold both `atmf006.nc` and
  `cubed_sphere_grid_atmf006.nc`, both keyed to `log.atm.f006`. The worker copies
  *all* files in the group (data first) then writes that rank's one marker.
- All four ranks wait on the **same** `log.atm.f006` but each writes its **own**
  per-product marker (`...log.atm.atmf.f006.txt`, `...sfcf...`, etc.).
- The **barrier rank** then waits for all four per-product markers for that hour
  and writes the single combined marker `log.f006.txt`. That combined marker is
  what downstream jobs actually key off.

```
log.atm.f006 (model)
│ (all four ranks wait on it)
├─ atmf rank  → copies atmf/csg_atmf  → writes ...atmf.f006.txt ─┐
├─ sfcf rank  → copies sfcf/csg_sfcf  → writes ...sfcf.f006.txt ─┤
├─ grib rank  → copies GFSPRS         → writes ...grib.f006.txt ─┼─▶ barrier ─▶ log.f006.txt
└─ flux rank  → copies GFSFLX         → writes ...flux.f006.txt ─┘   (combined)
```

ATM is always Scenario 1 (real sentinel). No size check. Its one Scenario 3 row is
the `fcst_finalized` marker (a group of one).

### Ocean (OCN, MOM6) — date-keyed sentinels

MOM6 names its sentinel by **valid date/time**, the same reference as its data
file — so sentinel and data always line up. Group size is effectively one row per
period. It uses Scenario 1 normally, and falls back to Scenario 2 for the final
period (whose sentinel is never produced) — which is why the history-done fallback
exists for ocean.

### Ice (CICE) — offset sentinels + an IC trigger

Ice is the awkward one:

- Each periodic hour waits on its own `log.ice.fHHHH` sentinel — **but** CICE
  labels that sentinel on the *model* clock, which (with IAU) is offset from the
  workflow forecast hour. If the expected name never appears, ice falls into
  Scenario 2 (history-done fallback + size check). This offset is the root of a
  real bug where a short file was published without being flagged.
- The **initial-condition (IC)** file has no sentinel of its own, so its row uses
  the **first periodic `.nc`** as a Scenario-3 data-file trigger: by the time that
  first periodic file appears, the IC has long been fully written, so it's safe to
  copy. The manager copies `iceh_ic.nc`, writes a synthetic note, and never copies
  the trigger.

Note ice is **not** a "hour N triggered by hour N+1" chain — the only forward
reference is the IC pointing at the first periodic file. Each ordinary hour has
its own sentinel (or the fallback).

---

## Section 8 — Restart safety

If the job crashes and restarts, the manager must not re-copy work already done.
Before copying any row it checks: **is the COM "done" marker already there?** If
so, it marks that row (and every row sharing the same sentinel) as done and moves
on. Because the marker is only ever written *after* the data is fully copied, its
presence is a reliable "this was already finished" signal.

---

## Mental model to lock in

- The manager copies model output to COM **as it's produced**, gated by
  **sentinels** ("done" notes).
- Each table row is: *data file, sentinel, COM data, COM marker.* Always **copy
  data first, write the marker last.**
- There are **three ways to create the COM marker**: copy the real sentinel
  (normal), or write a synthetic note for one of two reasons — the sentinel was
  missing but history is done (**ocn/ice only**), or the wait-on file was a
  stand-in trigger (**by name, any component**).
- **ATM** groups files under one sentinel and fuses per-product markers via a
  **barrier**; **OCN** sentinels are date-keyed and line up cleanly; **ICE**
  sentinels are offset (hence the fallback) and its IC rides on the first periodic
  file.
- The Scenario-2 **size check is comparative**, so it can't validate the *first*
  file of a type — a known blind spot.

---

### Source files (global-workflow)

- `scripts/exglobal_forecast_manager.sh` — builds the MPMD command list, launches
  the ranks, checks for failures.
- `ush/forecast_manager.sh` — the per-component worker: poll, copy, write markers.
- `ush/forecast_atm_barrier.sh` — fuses ATM per-product markers into the combined
  per-hour marker.
- `ush/run_mpmd.sh` — launches the ranks in parallel (srun / mpiexec).
- `ush/forecast_postdet.sh` — builds the product tables (ATM / OCN / ICE rows).
