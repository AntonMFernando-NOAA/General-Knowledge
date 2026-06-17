# Chapter 2.3 — Resuming Work on an ecFlow Suite

> *You set up a suite yesterday. You log back in today.
> What's still alive? What's broken? What do you need to redo?
> This chapter is the daily-routine checklist.*

---

# Section 1 — The 60-second daily startup

```bash
# 1. SSH to the SAME login node where you started the server
ssh -Y dlogin01

# 2. Load your saved environment
source ~/ecflow_c96.env

# 3. Sanity check: is the server alive?
ecflow_client --ping

# 4. Sanity check: is the server actually running (not HALTED/SHUTDOWN)?
ecflow_client --get_state | grep server_state

# 5. Sanity check: are my suites still loaded?
ecflow_client --get_state | grep -E "^suite "
```

If all four checks pass cleanly, you're done — open the GUI or start querying tasks.

If any check fails, jump to the matching section below.

---

# Section 2 — Recovery checklist (in order)

Walk this list top-to-bottom every morning. Each step is independent and only takes a few seconds.

## 2.1 Wrong login node

ecFlow servers are pinned to the host they were started on. If you started on `dlogin01` and SSH puts you on `dlogin04`, all your local `ecflow_client` calls will fail or talk to the wrong server.

```bash
hostname           # which node am I on?
echo $ECF_HOST     # which node does my env expect?
```

If they differ, hop to the right node:

```bash
ssh -Y dlogin01
source ~/ecflow_c96.env
```

Pin SecureCRT to the specific node so this never happens again. See Chapter 2.1, Section 1.1.

## 2.2 `--ping` fails

Server is dead. Restart it:

```bash
mkdir -p "${ECF_HOME}"          # cheap; harmless if it already exists
ecflow_start.sh -p "${ECF_PORT}" -d "${ECF_HOME}"
ecflow_client --ping            # should now succeed
```

`ecflow_start.sh` reads the latest `*.check` file in `ECF_HOME` and restores all suites + their state. Tasks that were `submitted` or `active` come back as-is; ecFlow does not re-run them.

## 2.3 `server_state` is `SHUTDOWN` or `HALTED`

The server is alive but not processing anything (no job submission, no task transitions, no checkpointing).

```bash
ecflow_client --restart    # SHUTDOWN/HALTED → RUNNING
ecflow_client --get_state | grep server_state
# expected: server_state:RUNNING
```

A server enters `HALTED` automatically when checkpoint writes fail (see 2.4). Fix the underlying issue first or it'll halt again.

## 2.4 Checkpoint error in `--get_state`

Look for `ECF_CHECKPT_ERROR` near the top of `--get_state` output. If it's present, the server can't save state and will halt itself. Almost always caused by a missing `ECF_HOME`:

```bash
mkdir -p "${ECF_HOME}"
ecflow_client --check_pt        # force a checkpoint to verify the dir works
ecflow_client --restart         # if the server halted itself
```

## 2.5 Suite not loaded (`--get_state | grep ^suite` is empty)

A clean checkpoint without any suites means either nothing was loaded yet, or the checkpoint was wiped. Load it fresh — see Section 3.

## 2.6 Suite is loaded but in `aborted` state

Tasks failed overnight. Don't reload — investigate first:

```bash
# Find the aborted tasks:
ecflow_client --get_state | grep -B2 "state:aborted" | head -40

# Inspect a specific one:
ecflow_client --query state /gfs_c96/primary/12/gdas/forecast/jgdas_fcst

# Read the job log:
cat ${ECF_HOME}/gfs_c96/primary/12/gdas/forecast/jgdas_fcst.<num>
```

Once you understand why it failed, either fix the script and rerun the single task, or reload the whole suite (Section 3).

To rerun one aborted task without reloading:

```bash
ecflow_client --requeue=force /gfs_c96/primary/12/gdas/forecast/jgdas_fcst
```

## 2.7 GUI won't start (Qt / xcb / `$DISPLAY` errors)

```
qt.qpa.plugin: Could not load the Qt platform plugin "xcb"
```

means X11 isn't reaching your terminal. None of the ecflow CLI commands need X11 — work from CLI for now and fix display afterwards.

```bash
echo $DISPLAY      # empty → no forwarding; reconnect with ssh -Y
xclock &           # smoke test: does any X app work?
```

If `$DISPLAY` is empty:

- Reconnect with `ssh -Y` (trusted forwarding).
- Verify SecureCRT has X11 forwarding enabled.
- If you SSH'd through a second hop (e.g. dlogin04 → dlogin01), redo with `ssh -Y dlogin01`.

See Chapter 2.1, Section 1.1 ("Set up X11 forwarding") for the full setup.

---

# Section 3 — Starting fresh on a new (or reset) suite

Use this when you want a clean slate: you edited the def file, you want to test from scratch, or last night's run is hopelessly tangled.

## 3.1 Decide: new suite, or wipe and reload?

| Scenario | Command |
|----------|---------|
| First time loading this suite | `--load` |
| Suite already on server, want a clean reset | `--delete=force=yes`, then `--load` |
| Suite is fine, just want to rerun aborted tasks | `--requeue` (no delete needed) |
| Suite definition changed | `--replace` (preserves variable overrides where possible) |

When in doubt, go with delete-and-reload. It's the simplest mental model.

## 3.2 Clean-slate sequence (the safe pattern)

```bash
# 0. Make sure the server is up and RUNNING (Section 2 above):
source ~/ecflow_c96.env
ecflow_client --ping
ecflow_client --get_state | grep server_state

# 1. Delete the existing suite (silent if it's not there):
ecflow_client --delete=force=yes /gfs_c96 2>/dev/null

# 2. Load the fresh def file (always use ABSOLUTE paths):
cd "${HOMEgfs}"
ecflow_client --load=dev/ecf/c96/defs/gfs_c96.def

# 3. SUSPEND the suite root before doing anything else:
ecflow_client --suspend=/gfs_c96

# 4. Verify the suite tree loaded as you expected:
ecflow_client --suites
ecflow_client --get_state /gfs_c96 | head -20

# 5. Re-apply your variable overrides:
bash dev/ecf/c96/bootstrap.sh
# OR run the per-variable --alter commands manually (Chapter 2.1 §1.6)

# 6. Verify the critical paths resolved:
ecflow_client --query variable /gfs_c96/primary:ECF_INCLUDE
ecflow_client --query variable /gfs_c96/primary:ECF_FILES
# Both must print absolute /lfs/h2/emc/... paths.

# 7. Resume only what you want to run first, then begin:
ecflow_client --suspend=/gfs_c96/primary/00     # keep 00Z parked
ecflow_client --resume=/gfs_c96
ecflow_client --resume=/gfs_c96/primary/12      # release 12Z first
ecflow_client --begin=gfs_c96
```

**Why suspend before begin?** `--begin` flips the suite from "queued" to "running" and ecFlow immediately starts submitting tasks whose triggers are satisfied. If `ECF_FILES` or `ECF_INCLUDE` is wrong, those jobs go straight to `aborted`. Suspending first lets you verify variables and inspect the tree before anything actually runs.

## 3.3 Loading a brand-new suite (different name)

Same as 3.2 but skip the `--delete` step:

```bash
source ~/ecflow_c96.env
cd "${HOMEgfs}"
ecflow_client --load=ecf/defs/gfs_v17_test.def
ecflow_client --suspend=/gfs_v17_test
# (apply variable overrides — bootstrap or --alter)
ecflow_client --resume=/gfs_v17_test
ecflow_client --begin=gfs_v17_test
```

You can have multiple suites loaded at once — they don't interfere.

## 3.4 Common load-time failures

```
LoadDefsCmd::LoadDefsCmd. Failed to parse file ecf/defs/gfs_v17_test.def
DefsStructureParser::DefsStructureParser: Unable to open file!
```
You ran `--load` from a directory that doesn't contain that relative path. Use the absolute path or `cd` to `${HOMEgfs}` first.

```
Add Suite failed: A Suite of name 'gfs_c96' already exists
```
Suite is loaded already. Either skip the load step, or delete first:
```bash
ecflow_client --delete=force=yes /gfs_c96
```

```
Expression node tree references failed for 'trigger ../fcst == complete'
Unrecognised path ../fcst for Node /gfs_c96_cycle/primary/12/gdas/atmos_prod
```
Trigger paths in your def file are wrong. ecFlow resolves family-level triggers relative to the family's PARENT, not the family itself. Use bare names (`fcst`) instead of `../fcst` for sibling references from a child family.

```
Begin failed as suite 'gfs_v17_test' is not loaded.
```
You ran `--begin` before `--load` succeeded. Re-run the load step and check the error.

## 3.5 Common preprocessing failures (after `--begin`)

These show up in `ecflow_client --get_state /path/to/task` between the `abort<:` and `>abort` markers. Tasks go to `aborted` immediately without ever reaching PBS.

```
EcfFile::create_job: Failed preprocessing :
TASK:/.../jgfs_stage_ic :
path/cmd(.../jgfs_stage_ic.ecf):
Could not open include file: /lfs/h2/emc/global/noscrub/USER/ecflow_c96/head.h
```

ecFlow 5.6's preprocessor searches `${ECF_HOME}` for `%include <head.h>` — **not** `${ECF_INCLUDE}`, even when `ECF_INCLUDE` is set on the suite. The fix is to put real copies of the include files in `${ECF_HOME}`:

```bash
cp ${HOMEgfs}/dev/ecf/<test>/include/head.h     "${ECF_HOME}/head.h"
cp ${HOMEgfs}/dev/ecf/<test>/include/tail.h     "${ECF_HOME}/tail.h"
cp ${HOMEgfs}/dev/ecf/<test>/include/envir-p1.h "${ECF_HOME}/envir-p1.h"
```

Use `cp`, not `ln -s`. Symlinks in `${ECF_HOME}` are not consistently honored by the preprocessor; real files always work.

```
Could not open include file: %HOMEgfs%/dev/ecf/c48_atm/include/head.h
```

`%HOMEgfs%` didn't expand. Either:
- `HOMEgfs` is not set as a sibling `edit` on the suite (unlikely if your bootstrap or inlined header runs).
- Nested variable expansion failed at preprocess time. Use **absolute paths** in `ECF_INCLUDE` and `ECF_FILES` rather than `%HOMEgfs%/...` so resolution doesn't depend on context. Build-time generators should bake these in:
  ```python
  HOMEgfs_ABS = str(REPO_ROOT)
  edits = [
      f"edit ECF_FILES   '{HOMEgfs_ABS}/dev/ecf/c48_atm/scripts'",
      f"edit ECF_INCLUDE '{HOMEgfs_ABS}/dev/ecf/c48_atm/include'",
  ]
  ```

## 3.6 Common J-job failures (after preprocessing succeeds)

When the task makes it past preprocessing, you'll see it move `queued` → `submitted` → `active`, and PBS will assign a job id (visible in `qstat`). Failures from here on appear in the `${ECF_HOME}<task>.<tryno>` log file rather than in the abort message itself.

```
FATAL ERROR: [.../ush/jjob_header.sh]: Unable to load config config.stage_ic
RETURN CODE 1
ABNORMAL EXIT
```

The J-job is looking for `${EXPDIR}/config.<jobname>` and not finding it. ecFlow's def file by itself does **not** populate the experiment directory — you have to run `setup_expt.py` (the same script Rocoto users run) to create the config files:

```bash
RUNTESTS=/lfs/h2/emc/global/noscrub/${USER}/c48_atm
mkdir -p "${RUNTESTS}/EXPDIR" "${RUNTESTS}/COMROOT"

python3 dev/workflow/setup_expt.py gfs forecast-only \
  --pslot c48_atm \
  --app ATM \
  --resdetatmos 48 \
  --idate 2021032312 \
  --edate 2021032312 \
  --comroot "${RUNTESTS}/COMROOT" \
  --expdir  "${RUNTESTS}/EXPDIR" \
  --yaml dev/ci/cases/yamls/gfs_defaults_ci.yaml \
  --overwrite
```

Then `EXPDIR` on the ecFlow suite (or the env you generate the def under) must point at `${RUNTESTS}/EXPDIR/<pslot>/`, where the configs landed.

```
ECF_HOST and ECF_PORT not reaching dev server
(task hangs in submitted forever, or qsub-side it looks like signal kill)
```

`module load prod_envir` inside `head.h` re-points `ECF_HOST`/`ECF_PORT` at NCO's production servers via the system `dhostfile`. The dev `head.h` must include this patch right before `ecflow_client --init`:

```bash
unset ECF_HOSTFILE
export ECF_HOST=%ECF_LOGHOST%
export ECF_PORT=%ECF_PORT%
```

Without it, the task's init/complete callbacks never reach your dev server.

```
killed by signal (likely via qdel)
```

Misleading. Often this is your job exiting from `set -e` after a script-level error in `head.h` or the J-job, not actually a `qdel`. Read `${ECF_HOME}<task>.<tryno>` — the FATAL ERROR line near the bottom names the real cause.

## 3.7 The four-layer failure progression

When a task aborts, the failure is always at one of four layers. Diagnose them in this order — fix the lowest layer first or the higher ones lie.

| Layer | Symptom | Look at | Common fix |
|-------|---------|---------|------------|
| 1. Connection | `Connection refused on dlogin01:34637` | `echo $ECF_PORT`, `ecflow_client --ping` | `unset ECF_HOSTFILE; source ~/ecflow_c96.env` |
| 2. Preprocessing | `Could not open include file:` (no `.job1` written) | `ecflow_client --get_state <task>` (`abort<:...>abort` block) | Copy include files into `${ECF_HOME}`; use absolute paths in `ECF_INCLUDE`/`ECF_FILES` |
| 3. PBS submission | Task `aborted`, no `.1` log, ecFlow says `qsub failed` | `${ECF_HOME}<task>.sub1` | Fix PBS resource directives (`-A`, `-q`, `-l select=...`) |
| 4. J-job execution | Task `aborted` with content in `.1`, often `FATAL ERROR` | `tail -50 ${ECF_HOME}<task>.<tryno>` | Run `setup_expt.py`, stage ICs, fix module loads |

A task only reaches layer N when layers 1..N-1 all succeed. So if layer 1 isn't right, you'll never see layer 2 errors — you'll see layer 1 errors over and over.

---

# Section 4 — Daily mistakes to avoid

A short list of every paper cut from real sessions. Read it once.

## 4.1 Don't run `--load` from `~/`

The repo path `ecf/defs/gfs_v17_test.def` only resolves under `${HOMEgfs}`. From your home directory it doesn't exist. Either `cd "${HOMEgfs}"` first, or always pass the absolute path.

## 4.2 Don't forget to `source ~/ecflow_c96.env`

Without it, `ECF_HOST`, `ECF_PORT`, and `HOMEgfs` are unset. `ecflow_client` then defaults to `localhost:3141` and silently talks to the wrong server (or no server at all). Make this the first thing every new terminal does.

## 4.3 Don't ignore `server_state:SHUTDOWN`

`--ping` succeeds against a SHUTDOWN server, so a green ping doesn't mean things are running. Always also check:

```bash
ecflow_client --get_state | grep server_state
```

If it's not `RUNNING`, your jobs aren't going anywhere.

## 4.4 Don't skip the variable-override step after `--load`

`--load` (and especially `--delete` + `--load`) wipes user-added variables. Tasks will run against whatever defaults are in the def file, which usually means wrong ECF_FILES, missing HOMEgfs, no PDY. Always run `bootstrap.sh` (or the manual `--alter` block) immediately after a fresh load.

## 4.5 Don't run `--begin` before verifying paths

```bash
ecflow_client --query variable /gfs_c96/primary:ECF_FILES
```

If this prints `/dev/ecf/c96/scripts` (no leading `/lfs/h2/...`), the variable wasn't expanded — `HOMEgfs` was empty when you loaded. Re-pin it with `--alter change variable` and re-query before resuming.

## 4.6 Don't expect ecflow_ui to work without `$DISPLAY`

The Qt xcb error means X11 isn't reaching your terminal. Reconnect with `ssh -Y` and verify `echo $DISPLAY` shows something like `localhost:10.0`. The CLI workflow (`--get_state`, `--query`) does everything the GUI does — use it while you debug X11.

## 4.7 Don't SSH-hop without re-forwarding X11

If SecureCRT lands you on `dlogin04` and you `ssh dlogin01` to reach the server, X11 doesn't follow that second hop unless you explicitly request it:

```bash
ssh -Y dlogin01
```

Or set `ForwardX11 yes` in `~/.ssh/config` on WCOSS2.

## 4.8 Don't edit a def file while the suite is running

The on-disk file and the in-memory suite are independent after `--load`. Editing the def file does nothing until you reload. To pick up changes:

```bash
ecflow_client --suspend=/gfs_c96
ecflow_client --replace /gfs_c96 dev/ecf/c96/defs/gfs_c96.def
ecflow_client --resume=/gfs_c96
```

Or do a full delete-and-reload (Section 3.2) if `--replace` complains.

## 4.9 Don't trust `find` output on `ECF_HOME` for "did the job run?"

`ECF_HOME` is the server's working directory, not the actual run dir. Job stdout/stderr lands in `${ECF_HOME}/<suite>/<family>/<task>.<tryno>`. The actual data products (model output, COM files) live under `DATAROOT` and `COMROOT`, set in your bootstrap. Check both.

---

## 4.10 Don't symlink `head.h` into `${ECF_HOME}`

ecFlow 5.6 follows symlinks inconsistently for `%include <name>` resolution. Use `cp` to put real copies in `${ECF_HOME}` instead. Symlinks may work today and silently break on the next reload.

## 4.11 Don't reuse another test's `include/` directory

Each test directory (`dev/ecf/c48_atm/`, `dev/ecf/c96/`, etc.) should own its own `include/head.h` and friends. If test A points at test B's include dir and someone deletes test B, test A breaks for a non-obvious reason. Keep them self-contained even if it duplicates a few files.

## 4.12 Don't expect a fresh def to populate `${EXPDIR}`

ecFlow's `--load` only registers the suite tree on the server. The J-jobs need `${EXPDIR}/config.<jobname>` files which only `setup_expt.py` (the Rocoto setup) creates. Run that **first** for the experiment, then point your ecFlow def at the same `${EXPDIR}`. Without it the first J-job aborts with `Unable to load config config.<jobname>`.

## 4.13 Don't trust the IDE-to-WCOSS2 sync blindly

If you edit a file in the IDE, verify it actually reached WCOSS2 before assuming changes took effect:

```bash
grep -c "<some marker from your edit>" path/to/file
```

When in doubt, deploy via `cat > file << 'EOF' ... EOF` directly on WCOSS2 to be sure.

# Section 5 — TL;DR cheat sheet

Pin this to your wall.

## Every morning

```bash
ssh -Y dlogin01
unset ECF_HOSTFILE
source ~/ecflow_c96.env
ecflow_client --ping
ecflow_client --get_state | grep -E "STATE|server_state"   # must imply RUNNING
ecflow_client --suites                                      # what's loaded?
```

## If server is dead

```bash
mkdir -p "${ECF_HOME}"
ecflow_start.sh -p "${ECF_PORT}" -d "${ECF_HOME}"
```

## If server is SHUTDOWN/HALTED

```bash
ecflow_client --restart
```

## First-time setup of a new test (one-time per experiment)

```bash
# 1. Populate EXPDIR with config files via Rocoto's setup tool:
RUNTESTS=/lfs/h2/emc/global/noscrub/${USER}/<test_name>
mkdir -p "${RUNTESTS}/EXPDIR" "${RUNTESTS}/COMROOT"
python3 dev/workflow/setup_expt.py gfs forecast-only \
  --pslot <test_name> --app ATM --resdetatmos 48 \
  --idate 2021032312 --edate 2021032312 \
  --comroot "${RUNTESTS}/COMROOT" --expdir "${RUNTESTS}/EXPDIR" \
  --yaml dev/ci/cases/yamls/gfs_defaults_ci.yaml --overwrite

# 2. Generate the def into EXPDIR:
EXPDIR="${RUNTESTS}/EXPDIR/<test_name>" python3 dev/ecf/<test_name>/build_def.py

# 3. Copy include files into ECF_HOME (preprocessing requirement):
cp dev/ecf/<test_name>/include/{head,tail,envir-p1}.h "${ECF_HOME}/"
```

## Load and run

```bash
SUITE=gfs_<test_name>
DEF="${RUNTESTS}/EXPDIR/<test_name>/${SUITE}.def"

ecflow_client --delete=force=yes "/${SUITE}" 2>/dev/null
ecflow_client --load="${DEF}"
ecflow_client --suspend="/${SUITE}"

# Verify variables look right BEFORE running:
ecflow_client --query variable "/${SUITE}/primary":HOMEgfs
ecflow_client --query variable "/${SUITE}/primary":ECF_FILES
ecflow_client --query variable "/${SUITE}/primary":ECF_INCLUDE

# Begin:
ecflow_client --resume="/${SUITE}"
ecflow_client --begin="${SUITE}"

# Watch:
sleep 15
ecflow_client --get_state "/${SUITE}" | grep -oE "state:[a-z]+" | sort | uniq -c
qstat -u "${USER}"
```

## When something aborts

```bash
TASK=/${SUITE}/<full/task/path>
ecflow_client --get_state "${TASK}"           # see abort<:...>abort message
ls -lt "${ECF_HOME}${TASK}".* 2>/dev/null
tail -50 $(ls -t "${ECF_HOME}${TASK}".[0-9]* | head -1)
```

Walk the four-layer failure progression (§3.7):
**connection → preprocessing → submission → J-job execution.**
Fix the lowest layer first.

## Open the GUI (if X11 is working)

```bash
ecflow_ui &
```
