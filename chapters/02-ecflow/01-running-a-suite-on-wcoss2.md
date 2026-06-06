# Chapter 1.2 — Running an ecFlow Suite on WCOSS2

> *Start a server. Load a suite. Begin a cycle. Watch it run.
> This chapter is the practical guide, the explanation, and the
> Rocoto comparison — one thing at a time.*

---

# Section 1 — Commands

Complete command sequence for running the C96 ecFlow suite on WCOSS2.
No prose. If you want to understand what these do, read Section 2.

## 1.1 One-time setup

### Save your environment

```bash
cat > ~/ecflow_c96.env <<'EOF'
unset ECF_HOSTFILE
module load ecflow
export ECF_HOST=dlogin01
export ECF_PORT=2137
export ECF_HOME=/lfs/h2/emc/global/noscrub/${USER}/ecflow_c96
export HOMEgfs=/lfs/h2/emc/global/noscrub/${USER}/global-workflow_gfsv17
EOF
```

### Install Xming on your Windows laptop (for the GUI)

Download and run the installer from https://sourceforge.net/projects/xming/
(user-space install; no admin rights needed).
Then in SecureCRT session config → Connection → Port Forwarding → Remote/X11
→ enable "Forward X11 packets".
Test: `echo $DISPLAY` should print `localhost:10.0` or similar after login.

## 1.2 Every new terminal on WCOSS2

```bash
source ~/ecflow_c96.env
ecflow_client --host=dlogin01 --port=2137 --ping
```

Expected: `ping server(dlogin01:2137) succeeded`.

## 1.3 Start the ecflow_server (once per WCOSS2 session; survives logout)

```bash
ssh dlogin01
source ~/ecflow_c96.env
mkdir -p "${ECF_HOME}"
ecflow_start.sh -p "${ECF_PORT}" -d "${ECF_HOME}"
ecflow_client --host=dlogin01 --port=2137 --ping
```

On subsequent logins the server is already running; just `--ping` to confirm.

Find the port if you forgot it:

```bash
ls /lfs/h2/emc/global/noscrub/${USER}/ecflow_c96/
# look for files named dlogin01.NNNN.check — NNNN is the port
```

## 1.4 Open the GUI (optional but useful)

Enable Xming on Windows, then:

```bash
source ~/ecflow_c96.env
ecflow_ui &
# In GUI: Servers → Manage Servers → Add server
# Name: c96   Host: dlogin01   Port: 2137
```

## 1.5 Load the suite

```bash
cd "${HOMEgfs}"
ecflow_client --host=dlogin01 --port=2137 --delete=force=yes /gfs_c96 2> /dev/null
ecflow_client --host=dlogin01 --port=2137 --load=dev/ecf/c96/defs/gfs_c96.def
ecflow_client --host=dlogin01 --port=2137 --suspend=/gfs_c96
ecflow_client --host=dlogin01 --port=2137 --suites
```

Expected: `gfs_c96`.

## 1.6 Bootstrap: set all suite variables

Run once after every `--load` (because `--delete` wiped them):

```bash
cd "${HOMEgfs}"
bash dev/ecf/c96/bootstrap.sh
```

This creates the dev workspace directories and applies all `--alter` overrides.

Or manually if you prefer to see every command:

```bash
ecflow_client --host=dlogin01 --port=2137 --alter add variable HOMEgfs        "${HOMEgfs}"      /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable ECF_LOGHOST    "${ECF_HOST}"     /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable ecflow_ver     5.6.0             /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable PDY            "$(date +%Y%m%d)" /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable PARATEST       NO                /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable COMPATH        " "                /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable MAILTO         " "                /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable DBNLOG         NO                /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable SENDDBN        NO                /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable SENDDBN_NTC    NO                /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable SENDCANNEDDBN  NO                /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable rrfs_ver       " "                /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable DATAROOT       "/lfs/h2/emc/global/noscrub/${USER}/c96_run/tmp" /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable COMROOT        "/lfs/h2/emc/global/noscrub/${USER}/c96_run/com" /gfs_c96

ecflow_client --host=dlogin01 --port=2137 --alter add variable ECF_JOB_CMD    "qsub %ECF_JOB% 1> %ECF_JOBOUT% 2>&1"           /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable ECF_KILL_CMD   "qdel %ECF_RID%"                                 /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --alter add variable ECF_STATUS_CMD "qstat %ECF_RID% > %ECF_JOB%.stat 2>&1"         /gfs_c96

ecflow_client --host=dlogin01 --port=2137 --alter change variable ECF_INCLUDE "${HOMEgfs}/dev/ecf/c96/include" /gfs_c96/primary
ecflow_client --host=dlogin01 --port=2137 --alter change variable ECF_FILES   "${HOMEgfs}/dev/ecf/c96/scripts" /gfs_c96/primary

# Verify paths resolved:
ecflow_client --host=dlogin01 --port=2137 --query variable /gfs_c96/primary:ECF_INCLUDE
ecflow_client --host=dlogin01 --port=2137 --query variable /gfs_c96/primary:ECF_FILES
```

Both should print full `/lfs/h2/emc/...` paths. If they start with `/dev/ecf/...` (no leading full path), `HOMEgfs` was empty during load — repin them.

## 1.7 Begin the run (12Z first, 00Z suspended)

```bash
ecflow_client --host=dlogin01 --port=2137 --suspend=/gfs_c96/primary/00
ecflow_client --host=dlogin01 --port=2137 --resume=/gfs_c96
ecflow_client --host=dlogin01 --port=2137 --resume=/gfs_c96/primary/12
ecflow_client --host=dlogin01 --port=2137 --begin=gfs_c96
qstat -u "${USER}"
```

## 1.8 Watch state

```bash
# ecFlow's view (run repeatedly):
ecflow_client --host=dlogin01 --port=2137 --get_state /gfs_c96/primary/12 \
  | grep -oE "state:[a-z]+" | sort | uniq -c

# PBS's view:
qstat -u "${USER}"
```

Healthy: counts shift from `queued` → `submitted` → `active` → `complete`.

## 1.9 Diagnose aborts

```bash
# Which tasks aborted and why:
ecflow_client --host=dlogin01 --port=2137 --get_state /gfs_c96/primary/12 \
  | grep "state:aborted" | head -5

# Job output file (for a specific task):
cat "${ECF_HOME}/gfs_c96/primary/12/gfs/prep/atmos/jgfs_atmos_prep_fsm.1"

# PBS output file (named <jobname>.o<jobid>; find recent ones):
ls -lt /lfs/h2/emc/global/noscrub/${USER}/ecflow_c96/*.o* | head -5

# Requeue after fixing:
ecflow_client --host=dlogin01 --port=2137 --requeue=force /gfs_c96
```

## 1.10 Reload after a code change

```bash
git pull origin feature/gfsv17-ecflow
ecflow_client --host=dlogin01 --port=2137 --delete=force=yes /gfs_c96
ecflow_client --host=dlogin01 --port=2137 --load=dev/ecf/c96/defs/gfs_c96.def
ecflow_client --host=dlogin01 --port=2137 --suspend=/gfs_c96
bash dev/ecf/c96/bootstrap.sh              # re-apply all variables
# then 1.7 again
```

## 1.11 Release 00Z (after 12Z completes)

```bash
# Confirm 12Z is fully done:
ecflow_client --host=dlogin01 --port=2137 --get_state /gfs_c96/primary/12 \
  | grep -oE "state:[a-z]+" | sort | uniq -c
# expect: N state:complete, nothing else

# Release:
ecflow_client --host=dlogin01 --port=2137 --resume=/gfs_c96/primary/00
```

## 1.12 Clean up

```bash
# Remove suite from server (files on disk untouched):
ecflow_client --host=dlogin01 --port=2137 --delete=force=yes /gfs_c96

# Stop the server entirely (only when fully done):
ecflow_client --host=dlogin01 --port=2137 --terminate=yes
```

---

# Section 2 — Explanations

This section explains what each command in Section 1 is doing and why it
has to be done that way.

Key files referenced in this chapter: [`gfs_c96.def`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/defs/gfs_c96.def) — [`bootstrap.sh`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/bootstrap.sh) — [`head.h`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/include/head.h) — [`tail.h`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/include/tail.h) — [`cycle_end.ecf`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/scripts/cycle_end.ecf)

## 2.1 Why there's a server at all

ecFlow keeps your workflow's state (which tasks are queued, running, done,
failed) in a long-running background process called `ecflow_server`. It
listens on a TCP port. All clients — the CLI tool `ecflow_client`, the GUI
`ecflow_ui`, and the ecflow calls inside running PBS jobs — talk to it over
that port.

Without a server, there's nowhere for state to live, nowhere for `--init`
and `--complete` callbacks from jobs to land, and no way to query what's
running.

Compare to Rocoto, which stores state in a SQLite file on disk and has no
server. Rocoto re-derives state from that file every 5 minutes when cron
invokes `rocotorun`. The tradeoff: Rocoto has no real-time GUI and no push
notifications when a task finishes; ecFlow has both.

## 2.2 What `ECF_HOST` and `ECF_PORT` are

Every client command has to know which server to talk to. Those two variables
are the host and the port. If they're not set correctly the client falls back
to the production `dhostfile` (`/lfs/h1/ops/prod/config/dhostfile`) which
points at NCO's operational servers on port 34637 — not yours.

That's why the env file starts with `unset ECF_HOSTFILE`: clear the fallback
before pointing the client at your server.

The `--host=dlogin01 --port=2137` flags in every command above are redundant
once you've `source ~/ecflow_c96.env`, but they're explicit, which removes
any ambiguity about which server you're hitting.

## 2.3 What the server checkpoint files are

The server writes its state to files in `ECF_HOME` at regular intervals.
The filenames are `<host>.<port>.check` and `<host>.<port>.ecf.log`. That's
how you find your port if you forget it. When the server restarts (after a
crash or a manual `terminate`/`start`), it reads those checkpoint files and
restores the suite's state.

## 2.4 What `--load` does

`--load` sends the `.def` file to the running server. The server parses it,
validates all triggers, and stores the suite tree in memory. Nothing runs yet.

The suite tree is a "loaded" copy in memory, not a live reference to the file
on disk. If you edit the `.def` file later, the server doesn't know. You have
to `--delete` and `--load` again to update it.

If `--load` fails with "Expression node tree references failed for trigger
../path/task", it means a trigger in the def points at a task or family that
doesn't exist in the suite. Fix the def and reload.

## 2.5 Why `--suspend` before `--begin`

`--begin` starts the suite running. Without `--suspend` first, the server
could start submitting jobs while you're still applying variable overrides in
the next step. Suspend freezes the whole suite so nothing runs until you say
`--resume`.

## 2.6 Why there are so many `--alter add variable` commands

ecFlow substitutes `%VAR%` placeholders in `.ecf` scripts at job-render time.
If `VAR` isn't defined on the suite (or any of its parent families), the
substitution fails and the task aborts immediately with:

```
EcfFile::variableSubstitution: failed: '%VAR%'
```

The `.def` file sets some variables at the suite level, but the production
`head.h` references many more that NCO's `prod_envir` module normally provides.
Running on a personal dev server means `prod_envir` is not available, so you
add them manually. `bootstrap.sh` does all of this in one run.

## 2.7 Why ECF_JOB_CMD must be set explicitly

By default, ecFlow runs job scripts inline (directly on the login node):

```
%ECF_JOB% 1> %ECF_JOBOUT% 2>&1
```

That means jobs run on `dlogin01` itself, which is prohibited and gets you an
admin warning. The WCOSS2 production ecFlow setup normally points this at a
queue-submission wrapper that calls `qsub`. On a dev server that automatic
override isn't present, so you set it manually:

```
ecflow_client --alter add variable ECF_JOB_CMD "qsub %ECF_JOB% 1> %ECF_JOBOUT% 2>&1" /gfs_c96
```

After that, ecFlow calls `qsub` for each ready task, which puts the job on the
compute cluster properly.

## 2.8 Why ECF_INCLUDE and ECF_FILES need absolute paths

The `.def` file sets these as:

```
edit ECF_INCLUDE '%HOMEgfs%/dev/ecf/c96/include'
edit ECF_FILES   '%HOMEgfs%/dev/ecf/c96/scripts'
```

ecFlow expands `%HOMEgfs%` at job-render time. If the server processes these
*before* you've added `HOMEgfs` with `--alter`, the substitution produces an
empty string and the paths become `/dev/ecf/c96/include` — which doesn't exist.

The safe fix: always explicitly pin them to absolute paths after loading:

```bash
ecflow_client --alter change variable ECF_INCLUDE "${HOMEgfs}/dev/ecf/c96/include" /gfs_c96/primary
```

Run `--query variable /gfs_c96/primary:ECF_INCLUDE` to confirm the value starts
with `/lfs/h2/...`.

## 2.9 Why `head.h` needs its own dev-local copy

The production `ecf/include/head.h` calls `module load prod_envir` inside the
PBS job on the compute node. That module re-points `ECF_HOST` and `ECF_PORT`
at the NCO production servers (`cdecflow01:34637`, etc.) by loading the
production `dhostfile`.

The job then calls `ecflow_client --init` to signal that it started. But with
the clobbered `ECF_HOST`/`ECF_PORT`, the call goes to NCO's servers, which
refuse the connection. The task stays stuck in `submitted` state indefinitely
even though the PBS job ran and finished.

The fix: a dev-local `dev/ecf/c96/include/head.h` that re-asserts the correct
host/port after all module loads, right before the `--init` call:

```bash
unset ECF_HOSTFILE
export ECF_HOST=%ECF_LOGHOST%
export ECF_PORT=%ECF_PORT%
timeout 300 ecflow_client --init=${ECF_RID}
```

The same re-pin is applied in `tail.h` before the `--complete` call.

## 2.10 What "submitted → active → complete" means

When ecFlow decides a task is eligible (all triggers satisfied), it:

1. Renders the `.ecf` script (variable substitution → `.job0` file).
2. Calls `ECF_JOB_CMD` (`qsub`) → PBS accepts the job: task goes to
   **submitted**.
3. The PBS job starts on a compute node, runs `head.h`, which calls
   `ecflow_client --init`. Server flips task to **active**.
4. The job runs the actual J-script (e.g. `JGLOBAL_FORECAST`).
5. Job reaches `tail.h`, which calls `ecflow_client --complete`. Server flips
   task to **complete** and re-evaluates all triggers. Downstream tasks that
   were waiting on this one become eligible.
6. If the job crashes before `tail.h`, the `trap "ERROR $?" ERR EXIT` in
   `head.h` calls `ecflow_client --abort`. Task goes to **aborted**. Dependent
   tasks don't start.

The state transition `submitted → (never active)` means PBS ran the job but
the `--init` callback never reached your server. Almost always this is the
`prod_envir`/dhostfile issue described in 2.9.

## 2.11 Common failure patterns

| What you see | Root cause | Fix |
|---|---|---|
| `failed: '%VAR%'` on load | Variable not set on suite | Add it with `--alter add variable` |
| `Directory ECF_FILES(%HOMEgfs%/...) does not exist` | Substitution failed before `HOMEgfs` was added | Pin absolute paths with `--alter change` |
| `submitted` but never `active`, PBS shows no job | `ECF_JOB_CMD` not set; ran inline on login node | Add `ECF_JOB_CMD qsub ...` |
| `submitted` but never `active`, PBS job did run | `prod_envir` clobbered `ECF_HOST`/`ECF_PORT` | Use dev-local [`head.h`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/include/head.h)/[`tail.h`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/include/tail.h) with re-pin |
| `exited with status 124` | 5-min inline run timeout on login node | Same: set `ECF_JOB_CMD` |
| `qsub: Error: Please include a valid walltime` | `.ecf` script missing PBS header | Every `.ecf` needs `#PBS -l walltime=...` etc. |
| Task `active` but PBS shows nothing, stuck forever | Job finished but `--complete` didn't reach server | Same dhostfile issue in `tail.h`; re-pin host/port |
| `No such file or directory: .../jobs/JGLOBAL_*` | `jobs/` symlink missing; J-scripts are in `dev/jobs/` | `ln -s dev/jobs jobs` at repo root (already committed) |
| `/lfs/f1/ops/prod/tmp: No such file or directory` | `DATAROOT` points at ops-only filesystem | Set `DATAROOT` to your noscrub area; `bootstrap.sh` does this |
| "Connection refused on port 34637" | `ECF_PORT` not set; client used production dhostfile | `unset ECF_HOSTFILE; export ECF_HOST=dlogin01; export ECF_PORT=2137` |

## 2.12 What to do when on a different login node

WCOSS2 round-robins logins to `dlogin01`–`dlogin04`. Your server is on
whichever node you started it from. From any other login node, you can still
reach it because all nodes share the internal network:

```bash
export ECF_HOST=dlogin01   # the node you started the server on
export ECF_PORT=2137
ecflow_client --ping
```

The `--ping` command connects *across* nodes with no issues.

If you genuinely need to move the server to a different node, stop it
(`--terminate=yes`) and `ecflow_start.sh` on the new node. Update
`~/ecflow_c96.env` to match.

## 2.13 What the GUI shows (ecflow_ui)

The GUI draws the suite as a colored tree:

- **Grey**: queued (waiting for dependencies)
- **Cyan**: submitted (in PBS queue)
- **Blue**: active (running on compute node)
- **Green**: complete
- **Red**: aborted

Right-click any task to view its generated job script, output log, variables,
or to requeue it. This is the equivalent of reading `rocotostat` output, but
interactive and real-time. You need Xming (Windows) or X11 forwarding to see
it. The CLI path works without it.

---

# Section 3 — Rocoto comparison

## 3.1 Architecture

```
Rocoto                                ecFlow
──────                                ──────
[ workflow.xml ]                      [ ecflow_server — always running ]
       │                                          │
       v  (every 5 min via cron)                  │ TCP port (e.g. 2137)
[ rocotorun  ]                            ┌───────┴───────┐
       │                                  v               v
[ gfs.db     ]                    [ ecflow_client ]  [ ecflow_ui ]
       │                                  │               │
       v                                  v               v
[ qsub/sbatch ]                       [ qsub (PBS) ]   (GUI, colored tree)
```

**Rocoto** is cron + a database. The engine runs for a few seconds every five
minutes. State lives in a SQLite file. No daemon, no port.

**ecFlow** is a database server + a GUI. A long-running daemon holds state in
RAM. Clients connect to it on demand. Push-notification updates, sub-second
GUI responses.

## 3.2 Concept mapping

| Concept | Rocoto | ecFlow |
|---|---|---|
| Whole workflow | XML file | suite (.def file) |
| Group of tasks | metatask | family |
| One job | task | task |
| Job script | `<command>` in XML | `.ecf` file under `ECF_FILES` |
| Time-stepped runs | `<cycledef>` (date range + stride) | repeat clauses, or one family per cycle |
| Dependency | `<taskdep .../>` | `trigger ../task == complete` |
| Variable in script | `<envar>` + shell `${VAR}` | `edit VAR 'value'` + `%VAR%` in `.ecf` |
| State storage | SQLite file (`gfs.db`) | server in-memory + checkpoint files |
| Engine startup | add to crontab (one line) | `ecflow_start.sh` (persistent daemon) |
| Status check | `rocotostat` | `ecflow_client --get_state` or `ecflow_ui` |
| Retry a task | `rocotorewind` + `rocotoboot` | `ecflow_client --requeue=force` |
| Freeze a task | not directly; remove from XML | `ecflow_client --suspend=/path/task` |
| Resume frozen | re-add to XML | `ecflow_client --resume=/path/task` |
| Config change at runtime | edit XML → `rocotorun` picks it up | `ecflow_client --alter change variable` |
| Reload workflow definition | automatic on next `rocotorun` | `--delete` + `--load` + re-apply `--alter` vars |
| Logs | `ROTDIR/logs/YYYYMMDDHH/<task>.log` | PBS `.o<jobid>` in submit dir; per-try `.1`, `.2`, ... in `ECF_HOME` |

## 3.3 Step-by-step comparison

### Setting up

| Step | Rocoto | ecFlow |
|---|---|---|
| Create experiment | `create_experiment.py -y case.yaml` | Load `.def` file into running server |
| Config files | `EXPDIR/<PSLOT>/config.*` (one per job category) | `edit VAR 'value'` in `.def` + `--alter` overrides |
| No equivalent | — | Start `ecflow_server` |

### Starting

| Step | Rocoto | ecFlow |
|---|---|---|
| Activate | Add crontab line | `ecflow_client --begin=<suite>` |
| Pause everything | Comment out crontab | `ecflow_client --suspend=/<suite>` |
| Resume | Uncomment crontab | `ecflow_client --resume=/<suite>` |

### Watching

| Step | Rocoto | ecFlow |
|---|---|---|
| Overall status | `rocotostat -d gfs.db -w gfs.xml` | `ecflow_client --get_state /<suite>` |
| One cycle | `rocotostat ... -c 202606040000` | `ecflow_client --get_state /<suite>/primary/12` |
| One task | `rocotostat ... -t taskname` | right-click in `ecflow_ui`, or `--get_state /path/task` |
| PBS jobs | `qstat -u $USER` | `qstat -u $USER` (same) |

### Fixing failures

| Scenario | Rocoto | ecFlow |
|---|---|---|
| Single task failed | `rocotorewind -t task -c cycle` then `rocotoboot` | `ecflow_client --requeue=force /path/task` |
| All tasks failed | re-run `rocotorun` with cron | `ecflow_client --requeue=force /<suite>` |
| Changed a J-script | nothing (read at next submit) | nothing (read at next submit) |
| Changed the XML/def | nothing (read on next `rocotorun`) | `--delete` + `--load` + re-apply `--alter` vars |
| Changed an `.ecf` script | N/A (scripts embedded in XML) | nothing (read at next submit) |

## 3.4 The things that don't exist on the other side

**Rocoto only:**
- `cycledef` groups (named cycle ranges with a stride; each task picks which groups it belongs to)
- `maxtries` / `cyclethrottle` / `taskthrottle` at the XML level
- SQLite database that you can inspect or back up

**ecFlow only:**
- `ecflow_server` daemon (nothing analogous in Rocoto)
- `ecflow_ui` GUI with colored tree
- `%VAR%` substitution inside job scripts (Rocoto passes vars as `<envar>`)
- Events (sub-states inside a task, e.g. "forecast reached f024" — downstream products can trigger off events)
- `suspend` / `resume` on individual families/tasks while the suite is running
- Port-based remote access from any machine that can reach the server node

## 3.5 Mental models side by side

> **Rocoto:** `rocotorun` is a visitor that comes every 5 minutes, reads the
> XML, checks the database, submits what's ready, and leaves. The state is
> always on disk and always readable even if the cron is stopped.

> **ecFlow:** `ecflow_server` is a resident that never leaves. It holds all
> state in RAM. Clients are visitors that stop by to ask questions or give
> orders, then leave.

The practical consequence:

- Rocoto: stop the cron → everything freezes safely. Nothing runs, nothing
  crashes. Resume by un-commenting the cron line.
- ecFlow: stop the server → all running jobs continue (they're in PBS already)
  but their `--complete` callbacks will fail (server is gone). Restart the
  server quickly to recover. The checkpoint files let it resume from where it
  was.

---

# Appendix — What each C96 suite file does

Quick reference for the files you interact with most. All live under
[`dev/ecf/c96/`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96).

### [`bootstrap.sh`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/bootstrap.sh)

Run once after every `--load`. Does three things:

1. Creates the three dev workspace directories the J-scripts write into:
   - `DATAROOT` → `/lfs/h2/emc/global/noscrub/$USER/c96_run/tmp`
   - `COMROOT`  → `/lfs/h2/emc/global/noscrub/$USER/c96_run/com`
   - `LOGROOT`  → `/lfs/h2/emc/global/noscrub/$USER/c96_run/logs`
2. Applies all suite-level `--alter add/change variable` overrides on the
   running server (ECF_LOGHOST, ECF_JOB_CMD, ECF_KILL_CMD, ECF_STATUS_CMD,
   HOMEgfs, ecflow_ver, PDY, all the production-style flags like SENDDBN=NO,
   and the dev workspace paths DATAROOT / COMROOT / LOGROOT).
3. Pins `ECF_FILES` and `ECF_INCLUDE` to absolute paths so variable
   substitution can't produce a broken partial path.

It is idempotent — safe to run multiple times.

### [`build_def.py`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/build_def.py)

The single source of truth for [`gfs_c96.def`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/defs/gfs_c96.def).
Reads the production [`ecf/defs/gfs_prod.def`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/ecf/defs/gfs_prod.def) and
produces a trimmed C96 version by:

- Keeping only the 12Z and 00Z cycle families (not 06Z / 18Z).
- Dropping cross-cycle trigger dependencies that would require a prior 06Z/18Z
  run to have completed (12Z is a cold-start).
- Capping per-forecast-hour events and product tasks at f120 (production runs
  to f384).
- Trimming the ENKF ensemble to 2 members (mem001, mem002) instead of 80.
- Rewriting the `cycle_end` trigger so 12Z → 00Z is the only handoff.

Do not edit `gfs_c96.def` by hand; run `python3 dev/ecf/c96/build_def.py`
to regenerate it after any change to the production def or to the builder.

### [`downscale_resources.py`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/downscale_resources.py)

Rewrites the `#PBS -l select=` and `#PBS -l walltime=` lines in 10 key `.ecf`
scripts to C96-appropriate values. Contains a hardcoded table of
production → C96 resource mappings:

| Script | Production | C96 |
|--------|-----------|-----|
| `jgfs_fcst.ecf` | 295 nodes / 6h | 1 node (82 ranks) / 3h |
| `jgdas_fcst.ecf` | 95 nodes / 1h50m | 1 node (82 ranks) / 20min |
| `jenkfgdas_fcst_master.ecf` | 8 nodes / 30min | 1 node (70 ranks) / 20min |
| `jgfs_atmos_anal.ecf` | 100 nodes | 4 nodes |
| `jgdas_atmos_anal.ecf` | 100 nodes | 4 nodes |
| `jgfs/jgdas_marine_analvar.ecf` | 8 nodes / 500GB | 1 node / 24GB |
| `jgfs/jgdas_marine_bmat.ecf` | 8 nodes / 500GB | 1 node / 24GB |
| `jenkfgdas_marine_ens_recenter.ecf` | 6 nodes / 45min | 1 node / 10min |

Run with `--check` to preview without writing. Run without flags to apply.
The output is committed alongside this script; re-run after any production
resource change that should be reflected in the C96 suite.

### [`gfs_c96.def`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/defs/gfs_c96.def)

The suite definition file. Generated by `build_def.py`; do not edit by hand.
Contains:

- One `suite gfs_c96` block with suite-level `edit` variable declarations
  (`HOMEgfs`, `ECF_FILES`, `ECF_INCLUDE`, `QUEUE='dev'`, etc.).
- Two cycle families: `12` (cold-start) and `00` (depends on 12Z).
- Full GFS / GDAS / ENKF family structure with triggers, events, and task
  definitions matching the production layout.

### [`dev/ecf/c96/include/head.h`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/include/head.h) and [`tail.h`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/include/tail.h)

Dev-local copies of the standard ecFlow job wrappers. Identical to the
production `ecf/include/head.h` / `tail.h` except for one patch: right before
the `ecflow_client --init` call (in `head.h`) and the `ecflow_client --complete`
call (in `tail.h`), they re-assert:

```bash
unset ECF_HOSTFILE
export ECF_HOST=%ECF_LOGHOST%
export ECF_PORT=%ECF_PORT%
```

This is necessary because `module load prod_envir` inside `head.h` silently
re-points `ECF_HOST` and `ECF_PORT` at NCO's production ecflow servers
(`cdecflow01:34637` etc.) via the system `dhostfile`. Without this patch, jobs
reach the compute node and run fine, but the `--init` and `--complete`
callbacks never reach your dev server — the task stays `submitted` forever.

### [`cycle_end.ecf`](https://github.com/AntonMFernando-NOAA/global-workflow/blob/feature/gfsv17-ecflow/dev/ecf/c96/scripts/cycle_end.ecf)

A C96-specific override of the production `cycle_end` script. The production
version cycles through 00Z → 06Z → 12Z → 18Z → 00Z indefinitely, advancing
the PDY and requeuing the next cycle. The C96 version only handles two cases:

- `CYC=12`: requeues only the 00Z `cycle_end`, triggering the second and
  final cycle.
- `CYC=00`: prints a completion message and exits cleanly (no next cycle).
- Any other value: exits with an error.

PBS resources: 1 node, 1 CPU, 1GB, 5-minute walltime.

