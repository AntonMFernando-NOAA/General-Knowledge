# Chapter 2.2 — Developing for ecFlow: Experiment Setup Compared to Rocoto

> *You know how to run a Rocoto experiment: create the EXPDIR, set config,
> generate the XML, add a crontab, check `rocotostat`. Now you need to do
> the same thing for ecFlow. This chapter maps every Rocoto concept you
> already know to its ecFlow equivalent — and points out where there is no
> equivalent.*

---

# Section 1 — Commands

Commands for setting up and running a development experiment under each
engine. No prose. See Section 2 for the why.

## 1.1 Rocoto: create a dev experiment

```bash
cd ${HOMEglobal}/dev/workflow

# Create experiment directories and config files
python setup_expt.py gfs cycled \
    --pslot myexpt \
    --idate 2026060400 \
    --edate 2026060412 \
    --expdir /scratch3/NCEPDEV/global/${USER}/RUNTESTS/EXPDIR \
    --comroot /scratch3/NCEPDEV/global/${USER}/RUNTESTS/COMROT

# Generate the Rocoto XML
python setup_workflow.py \
    /scratch3/NCEPDEV/global/${USER}/RUNTESTS/EXPDIR/myexpt \
    rocoto

# Add the crontab entry
crontab -e
# Paste the contents of EXPDIR/myexpt/myexpt.crontab

# Or run manually:
rocotorun -d EXPDIR/myexpt/myexpt.db -w EXPDIR/myexpt/myexpt.xml

# Check status:
rocotostat -d EXPDIR/myexpt/myexpt.db -w EXPDIR/myexpt/myexpt.xml
```

## 1.2 ecFlow (dev/C96): set up and run a dev experiment

There is no `setup_expt.py` equivalent for ecFlow dev. Instead:

```bash
# Step 1: clone and checkout
cd /lfs/h2/emc/global/noscrub/${USER}
git clone -b feature/gfsv17-ecflow \
    https://github.com/AntonMFernando-NOAA/global-workflow.git \
    global-workflow_gfsv17
cd global-workflow_gfsv17
export HOMEgfs=$PWD

# Step 2: make sure the suite def is up to date
python3 dev/ecf/c96/build_def.py

# Step 3: start your ecFlow server (once; see Chapter 2.1)
source ~/ecflow_c96.env
ecflow_start.sh -p "${ECF_PORT}" -d "${ECF_HOME}"
ecflow_client --ping

# Step 4: load the suite and bootstrap directories + variables
ecflow_client --load=[dev/ecf/c96/defs/gfs_c96.def](https://github.com/AntonMFernando-NOAA/global-workflow/blob/d17125ed3661468a5c4ff92bcb140173d3743b3d/dev/ecf/c96/build_def.py)
ecflow_client --suspend=/gfs_c96
bash [dev/ecf/c96/bootstrap.sh](https://github.com/AntonMFernando-NOAA/global-workflow/blob/d17125ed3661468a5c4ff92bcb140173d3743b3d/dev/ecf/c96/bootstrap.sh)

# Step 5: begin (12Z first)
ecflow_client --suspend=/gfs_c96/primary/00
ecflow_client --resume=/gfs_c96
ecflow_client --resume=/gfs_c96/primary/12
ecflow_client --begin=gfs_c96

# Step 6: check status
ecflow_client --get_state /gfs_c96/primary/12 \
  | grep -oE "state:[a-z]+" | sort | uniq -c
qstat -u "${USER}"
```

## 1.3 Where the directories are

### Rocoto

| Directory | Default location on Hera | Set by |
|---|---|---|
| EXPDIR | `/scratch3/NCEPDEV/global/${USER}/RUNTESTS/EXPDIR/myexpt/` | `--expdir` CLI arg |
| ROTDIR (= COMROOT) | `/scratch3/NCEPDEV/global/${USER}/RUNTESTS/COMROT/myexpt/` | `--comroot` CLI arg |
| DATAROOT | `${STMP}/RUNDIRS/myexpt/gfs.YYYYMMDDHH/` | Derived from `STMP` in host YAML |
| LOGDIR | `${ROTDIR}/logs/YYYYMMDDHH/` | Inside ROTDIR |

### ecFlow (dev/C96)

| Directory | Default location on WCOSS2 | Set by |
|---|---|---|
| HOMEgfs (= code root) | `/lfs/h2/emc/global/noscrub/${USER}/global-workflow_gfsv17/` | You set `HOMEgfs=$PWD` |
| DATAROOT | `/lfs/h2/emc/global/noscrub/${USER}/c96_run/tmp/` | `bootstrap.sh` |
| COMROOT | `/lfs/h2/emc/global/noscrub/${USER}/c96_run/com/` | `bootstrap.sh` |
| LOGROOT | `/lfs/h2/emc/global/noscrub/${USER}/c96_run/logs/` | `bootstrap.sh` |
| ECF_HOME (server files + job output) | `/lfs/h2/emc/global/noscrub/${USER}/ecflow_c96/` | `~/ecflow_c96.env` |
| EXPDIR equivalent | None — the def file IS the config | — |

---

# Section 2 — Explanations

## 2.1 The core difference in mental model

**Rocoto dev:** every experiment is self-contained. You create a slot (`PSLOT`),
it owns a directory tree (EXPDIR + ROTDIR), and it's independent of every other
experiment. You can have ten experiments running simultaneously without them
interfering.

**ecFlow dev:** there is no per-experiment directory. The suite definition
(`.def` file) is the configuration. There is one server running your suite.
To run a different experiment you change the `.def` file and reload, or run a
completely separate server on a different port.

This is intentional — ecFlow was designed for production where there's one
canonical suite and NCO operators manage it centrally.

## 2.2 What `setup_expt.py` / `create_experiment.py` produces for Rocoto

Running `setup_expt.py` creates:

```
EXPDIR/myexpt/
├── config.base        ← all path variables, dates, grid config
├── config.anal        ← analysis settings
├── config.fcst        ← forecast settings
├── config.com         ← COM path templates
├── config.*           ← one file per major job
├── myexpt.xml         ← the Rocoto workflow (auto-generated)
├── myexpt.crontab     ← the cron entry
└── myexpt.db          ← created by rocotorun on first invocation
```

The `config.*` files are sourced by every J-script. They're what translate
"this experiment" into specific paths, dates, and switches that the science
code reads.

Running `setup_workflow.py` generates `myexpt.xml` from those configs. It
embeds `ROTDIR`, `PSLOT`, and cycle ranges as XML entities at the top of the
file.

## 2.3 What the ecFlow equivalent is (and isn't)

There is no `setup_expt.py` for ecFlow. The equivalent is:

| Rocoto | ecFlow dev equivalent |
|---|---|
| `setup_expt.py` | `build_def.py` (generates the `.def` from the production def) |
| `config.base` (EXPDIR, paths, dates) | Suite-level `edit` variables in the `.def` + `--alter` overrides |
| `config.com` (COM path templates) | `compath.py` in ops; your `COMROOT` variable in dev |
| `setup_workflow.py` | — (the `.def` is already the workflow definition) |
| `myexpt.crontab` | `ecflow_start.sh` (one-time) + `--begin` |
| `rocotorun` cron invocation | The persistent `ecflow_server` daemon |

The key gap: Rocoto config files let you tune each job individually (walltime,
MPI ranks, queue, etc.) for a specific experiment. In ecFlow, those are inside
the `.ecf` scripts themselves. For the C96 dev suite, `downscale_resources.py`
plays the role that the config-override mechanism plays for Rocoto.

## 2.4 ROTDIR vs COMROOT

In Rocoto, model output lives at:

```
${ROTDIR}/${RUN}.${YYYYMMDD}/${HH}/model/atmos/history/
```

`ROTDIR` is `COMROOT/PSLOT`, set in `config.base`. Every task writes into
a subdirectory here.

In ecFlow dev (C96), you set `COMROOT` in `bootstrap.sh`. The jobs compute
their output paths from `compath.py` (or, in dev, from the `COMROOT` variable
you set). The structure is the same:

```
${COMROOT}/${RUN}.${YYYYMMDD}/${HH}/model/atmos/history/
```

They're the same pattern. The difference is where the root lives (your
scratch vs the ops `/lfs/h1/ops/prod/com/gfs/v17.0/`).

## 2.5 DATAROOT / RUNDIR

Both engines use a per-job working directory that's created at job start and
(by default) deleted at job end:

```
Rocoto:  ${STMP}/RUNDIRS/${PSLOT}/${RUN}.YYYYMMDDHH/   ← DATAROOT
ecFlow dev:  ${DATAROOT}/marineanalysis.YYYYMMDDHH/marinebmat/   ← DATA
```

In Rocoto, `DATAROOT` is passed to the job via a Rocoto `<envar>`. In ecFlow
dev, you set it with `--alter add variable DATAROOT` (or via `bootstrap.sh`).
The J-scripts compute `DATA` from `DATAROOT` — the same J-scripts, the same
pattern, regardless of which workflow engine ran them.

## 2.6 LOGDIR / logs

| | Rocoto | ecFlow dev |
|---|---|---|
| Per-job stdout/stderr | `${ROTDIR}/logs/YYYYMMDDHH/<task>.log` | `${ECF_HOME}/gfs_c96/primary/12/<task>.1` (try 1), `.2` (retry), etc. Also PBS `.o<jobid>` in `${ECF_HOME}/` |
| Workflow engine log | `myexpt.db` (SQLite, binary) | `${ECF_HOME}/dlogin01.2137.ecf.log` (text) |
| Status command | `rocotostat` | `ecflow_client --get_state` or `ecflow_ui` |

## 2.7 Making a code change: workflow for each engine

### Rocoto

```bash
# 1. Edit the science code (J-scripts, ex-scripts, etc.)
# 2. If you changed task definitions, re-run setup_workflow.py to regenerate XML
# 3. rocotorun picks up the new XML on the next cron tick
# 4. Changed J-scripts are read at each task's next submission automatically
```

### ecFlow dev

```bash
# 1. Edit the science code (J-scripts, ex-scripts, .ecf scripts, etc.)
# 2. If you changed .ecf scripts only: nothing — re-read at next submit
# 3. If you changed the .def (new task, new trigger, etc.):
git pull
python3 dev/ecf/c96/build_def.py        # regenerate from production def
ecflow_client --delete=force=yes /gfs_c96
ecflow_client --load=dev/ecf/c96/defs/gfs_c96.def
ecflow_client --suspend=/gfs_c96
bash dev/ecf/c96/bootstrap.sh           # re-apply all variable overrides
# then --begin again
```

The ecFlow reload is more steps than Rocoto because the server holds an
in-memory copy of the def. Changes on disk only take effect after a reload.

## 2.8 How the C96 suite bridges the two worlds

The C96 suite is deliberately designed to let developers test ecFlow-side
changes using a Rocoto-like dev workflow:

- **Code lives in the repo** (like Rocoto), not in an installed production package
- **Resources are downsized** to C96 so it runs in minutes on the dev queue (like the C96 Rocoto CI cases)
- **Two cycles only** (12Z + 00Z) instead of a continuous round-robin
- **bootstrap.sh** fills the role of `setup_expt.py`: creates dirs, sets variables, gets you to the point where `--begin` works

The gap compared to Rocoto: there's no per-experiment isolation. If you want
to compare two versions of the suite you need two separate server ports or
two separate suite names.

---

# Section 3 — Side-by-side reference

## 3.1 Full concept map

| Concept | Rocoto dev | ecFlow dev (C96) |
|---|---|---|
| Code root | `HOMEglobal` = repo checkout | `HOMEgfs` = repo checkout |
| Experiment slot name | `PSLOT` | No equivalent (suite name is the slot) |
| Experiment config dir | `EXPDIR/<PSLOT>/` | No separate dir; suite `edit` variables in `.def` |
| Setup script | `create_experiment.py` / `setup_expt.py` | `bootstrap.sh` (partial equivalent) |
| Workflow definition | `<PSLOT>.xml` (generated by `setup_workflow.py`) | `gfs_c96.def` (generated by `build_def.py`) |
| COM / model output root | `ROTDIR` = `COMROOT/<PSLOT>` | `COMROOT` set in `bootstrap.sh` |
| Per-cycle COM subdir | `ROTDIR/<RUN>.YYYYMMDD/HH/` | `COMROOT/<RUN>.YYYYMMDD/HH/` (same) |
| Job working dir | `${STMP}/RUNDIRS/<PSLOT>/<RUN>.YYYYMMDDHH/` | `${DATAROOT}/.../<RUN>.YYYYMMDDHH/` |
| Engine startup | Add crontab line | `ecflow_start.sh` + `--begin` |
| Config change | Edit `config.*` in EXPDIR, re-run `setup_workflow.py` | `ecflow_client --alter change variable` or reload def |
| Resource tuning | `config.resources.*` per experiment | Hardcoded in `.ecf` scripts; `downscale_resources.py` |
| Task script | J-script path in `<command>` in XML | `.ecf` file under `ECF_FILES` |
| Variable injection into job | Rocoto `<envar>` → shell `${VAR}` | ecFlow `edit VAR 'value'` → `%VAR%` in `.ecf` |
| Status check | `rocotostat` | `ecflow_client --get_state` or `ecflow_ui` |
| Retry a task | `rocotorewind` + `rocotoboot` | `ecflow_client --requeue=force /path/task` |
| Logs | `ROTDIR/logs/YYYYMMDDHH/<task>.log` | `ECF_HOME/gfs_c96/primary/12/<task>.1` |
| Workflow log | `myexpt.db` (SQLite) | `ECF_HOME/dlogin01.2137.ecf.log` |

## 3.2 Where the pain is, and why

**Rocoto** is optimized for researchers: easy experiment isolation (`PSLOT`),
trivial config override (edit a file), no server to manage. The tradeoff:
no real-time monitoring, polling-based (5-min lag), no GUI.

**ecFlow dev** is painful to set up the first time because it was not designed
for per-developer experiment isolation. It was designed for a single canonical
production suite managed by a team. The dev use of ecFlow involves manually
recreating things (`DATAROOT`, `prod_envir` vars, path pins) that production
gets automatically.

Once you've done it once and codified it in `bootstrap.sh`, subsequent runs
are much faster. But the initial friction is real.

## 3.3 Why bother with ecFlow dev at all

Because the thing you're testing is the ecFlow-specific logic:

- Do the triggers between tasks fire in the right order?
- Does the cycle handoff in `cycle_end.ecf` work correctly?
- Does a new task you added sit in the right place in the dependency tree?
- Do events (sub-task signals like "forecast reached f024") release
  downstream tasks at the right time?

None of those can be tested with Rocoto, because Rocoto doesn't have them.
The C96 dev ecFlow suite exists so that those tests can be done cheaply
(small resolution, 2 cycles) before going to the full production suite.
