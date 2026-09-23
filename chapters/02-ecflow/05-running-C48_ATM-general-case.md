# Chapter 2.5 — Running the C48_ATM Case with ecFlow (General Case)

> *The C48_ATM forecast-only case is the simplest ecFlow test case in
> global-workflow. This chapter walks through running it on Ursa from
> first connection to a complete suite. The same steps apply to other
> cases and platforms with minor path adjustments.*

---

# Section 1 — Commands

Complete command sequence for running the C48_ATM ecFlow case on Ursa.
No prose. If you want to understand what these do, read Section 2.

## 1.1 Prerequisites

- NOAA RDHPCS account with Ursa access
- PuTTY (Windows) or SSH client (Mac/Linux)
- Global-workflow repo checked out and built on Ursa
- Slurm scheduler access for job submission

## 1.2 Connect to Ursa

### From Windows (PuTTY)

1. Open PuTTY
2. Hostname: `ursa.rdhpcs.noaa.gov`
3. Port: `22`
4. Connection type: SSH
5. Click **Open**
6. Login with your RDHPCS username and RSA token + PIN

To save for future use: in the **Session** panel, type a name (e.g. `Ursa`)
in "Saved Sessions" and click **Save**. Double-click the saved session next
time.

### From Mac/Linux

```bash
ssh <username>@ursa.rdhpcs.noaa.gov
```

### X11 forwarding for ecflow_ui (GUI)

`ecflow_ui` requires X11 forwarding.

**PuTTY:** Connection → SSH → X11 → check "Enable X11 forwarding".
Install [VcXsrv](https://sourceforge.net/projects/vcxsrv/) or
[Xming](https://sourceforge.net/projects/xming/) as your local X server.

**Mac/Linux:**
```bash
ssh -X <username>@ursa.rdhpcs.noaa.gov
```

Verify X11 after login:
```bash
xterm &    # a small terminal window should appear on your screen
```

### After logging in

```bash
hostname   # expected: ulogin01 or similar
cd /scratch3/NCEPDEV/global/${USER}
```

## 1.3 Environment setup

Set these before running any ecFlow scripts. Add to `~/.bashrc` or source
in each session.

```bash
# Load the ecFlow module
module load ecflow

# Remove any stale host file
unset ECF_HOSTFILE

# Set the global-workflow repo path
export HOMEglobal=/scratch3/NCEPDEV/global/${USER}/global-workflow

# Choose an ecFlow server
# Each user runs their own ecFlow server on a unique port.
# Use your UID offset by 1500 to avoid collisions with other users:
export ECF_PORT=$(( $(id -u) + 1500 ))
export ECF_HOST=$(hostname)
echo "Your ecFlow server: ${ECF_HOST}:${ECF_PORT}"

# Set the ecFlow job directory
export ECF_HOME=/scratch3/NCEPDEV/global/${USER}/ecflow
mkdir -p "${ECF_HOME}"
```

Verify:
```bash
echo "ECF_HOST   = ${ECF_HOST}"
echo "ECF_PORT   = ${ECF_PORT}"
echo "ECF_HOME   = ${ECF_HOME}"
echo "HOMEglobal = ${HOMEglobal}"
ecflow_client --ping   # should say "ping ... succeeded"
```

## 1.4 ecFlow server

### Check if a server is already running

```bash
ecflow_client --ping
```

If it responds with `ping server(...) succeeded`, the server is up —
skip to §1.5.

### Find your server port

```bash
# Check for a server under your user
ecflow_client --host=$(hostname) --port=${ECF_PORT} --ping

# Or find all ecflow_server processes on this host
ps -u ${USER} -f | grep ecflow_server
# The --port value in the output is your ECF_PORT
```

### Start your own server

```bash
# 1. Pick a unique port (UID + 1500 avoids collisions)
export ECF_PORT=$(( $(id -u) + 1500 ))
echo "Starting ecFlow server on port ${ECF_PORT}"

# 2. Create the job directory
export ECF_HOME=/scratch3/NCEPDEV/global/${USER}/ecflow
mkdir -p "${ECF_HOME}"

# 3. Start the server
ecflow_start.sh -p ${ECF_PORT} -d ${ECF_HOME}

# 4. Verify
export ECF_HOST=$(hostname)
ecflow_client --ping
# Expected: ping server(<hostname>:<port>) succeeded in 00:00:00.00...

# 5. Save for future sessions
echo "Add to your ~/.bashrc:"
echo "  export ECF_HOST=${ECF_HOST}"
echo "  export ECF_PORT=${ECF_PORT}"
echo "  export ECF_HOME=${ECF_HOME}"
```

### Stop the server (only when completely done)

```bash
ecflow_client --halt=yes       # stop scheduling
ecflow_client --check_pt       # save server state
ecflow_client --terminate=yes  # shut down the server process
```

## 1.5 Run the C48_ATM case

### Quick start (all defaults)

```bash
cd ${HOMEglobal}
python3 dev/workflow/ecflow/c48_atm_ecflow.py
```

Answer `y` to the cleanup and delete prompts. The suite is loaded but
**not started**. The script prints the `ecflow_client --begin` command
to run when you are ready.

### With custom paths

```bash
python3 dev/workflow/ecflow/c48_atm_ecflow.py \
    --pslot my_C48_test \
    --comroot /scratch4/NCEPDEV/stmp/${USER}/COMROOT \
    --expdir /scratch3/NCEPDEV/global/${USER}/EXPDIR \
    --stmp /scratch4/NCEPDEV/stmp/${USER}
```

### CLI options

| Option | Description |
|--------|-------------|
| `--yaml PATH` | Override the case YAML file |
| `--pslot NAME` | Override experiment name (default: `C48_ATM_ecflow`) |
| `--comroot PATH` | Override output data directory |
| `--expdir PATH` | Override experiment config directory |
| `--stmp PATH` | Override runtime scratch directory |
| `--suite-name NAME` | Override ecFlow suite name |
| `--overwrite` | Overwrite a previously created experiment |

### What the script does behind the scenes

1. Cleans up any previous run directories (with prompt)
2. Creates the experiment via `setup_expt`
3. Generates the `.def` file and copies `.ecf` scripts
4. Loads the suite into the ecFlow server (prompts if it already exists)

The suite is loaded but **not started**. The script prints the
`ecflow_client --begin` command to run when you are ready.

## 1.6 Monitor the run

### ecflow_ui (GUI)

```bash
ecflow_ui &
```

Connect to `${ECF_HOST}:${ECF_PORT}`. The suite tree shows task states:
queued (blue), submitted (cyan), active (green), complete (yellow),
aborted (red).

### Command line

```bash
# Suite status overview
ecflow_client --get_state /C48_ATM_ecflow

# Watch a specific task
ecflow_client --get_state /C48_ATM_ecflow/gfs/2021032312/fcst

# View job output for a task
cat ${ECF_HOME}/fcst.1    # .1 = first try number
```

### Log files

Task logs are written to `{ROTDIR}/logs/`:
```bash
ls ${COMROOT}/C48_ATM_ecflow/logs/
```

## 1.7 Common operations

### Rerun a failed task

```bash
# In ecflow_ui: right-click task → Rerun
# Or from CLI:
ecflow_client --force=queued /C48_ATM_ecflow/gfs/2021032312/fcst
```

### If a task fails (red/aborted)

```bash
# 1. Check the job output for the error
cat ${ECF_HOME}/<task_name>.1

# 2. Fix the issue (edit .ecf, fix paths, etc.)

# 3. Rerun the task
ecflow_client --force=queued /C48_ATM_ecflow/gfs/2021032312/<task_name>
```

### Suspend / resume the suite

```bash
ecflow_client --suspend /C48_ATM_ecflow
ecflow_client --resume /C48_ATM_ecflow
```

### Delete the suite

```bash
ecflow_client --suspend /C48_ATM_ecflow
ecflow_client --kill /C48_ATM_ecflow
sleep 5
ecflow_client --delete=force yes /C48_ATM_ecflow
```

### Update .ecf scripts without regenerating the .def

```bash
bash dev/workflow/ecflow/sync_ecf_scripts.sh \
    ${EXPDIR}/C48_ATM_ecflow/ecf_scripts
```

### Regenerate the .def from scratch (rerun the whole suite)

```bash
python3 dev/workflow/ecflow/c48_atm_ecflow.py --overwrite
```

### Clean up after a run

```bash
# Delete the suite from the server
ecflow_client --suspend /C48_ATM_ecflow
ecflow_client --kill /C48_ATM_ecflow
sleep 5
ecflow_client --delete=force yes /C48_ATM_ecflow

# Optionally remove the experiment directories
rm -rf ${RUNTESTS}/EXPDIR/C48_ATM_ecflow
rm -rf ${RUNTESTS}/COMROOT/C48_ATM_ecflow
```

## 1.8 Day-to-day workflow (quick reference)

Once the one-time setup (§1.2–1.4) is done, this is all you need each day.

### Starting a new session

```bash
# 1. SSH into Ursa
ssh -X <username>@ursa.rdhpcs.noaa.gov

# 2. Source your environment (or add to ~/.bashrc once)
module load ecflow
unset ECF_HOSTFILE
export ECF_PORT=$(( $(id -u) + 1500 ))
export ECF_HOST=$(hostname)
export ECF_HOME=/scratch3/NCEPDEV/global/${USER}/ecflow
export HOMEglobal=/scratch3/NCEPDEV/global/${USER}/global-workflow

# 3. Verify the server is alive
ecflow_client --ping
```

### Run the case

```bash
cd ${HOMEglobal}
python3 dev/workflow/ecflow/c48_atm_ecflow.py
```

### End of day

The ecFlow server persists across sessions. Log out and come back
tomorrow — the suite continues running (or waiting) on the server.
Re-source your environment variables when you reconnect.

---

# Section 2 — Explanations

## 2.1 What c48_atm_ecflow.py does

The script is the single entry point for running the C48_ATM forecast-only
test case. It chains together several steps that you would otherwise run
manually:

1. **Experiment creation** — calls `setup_expt` to produce the experiment
   config directory (EXPDIR) with all `config.*` files.
2. **Workflow generation** — calls `setup_workflow` which delegates to
   `ecflow_suite_factory` to build the `.def` file and copy `.ecf` scripts.
3. **Server load** — calls `ecflow_client --load` to push the `.def` into
   the running ecFlow server.

The suite is loaded but not started. The script prints the
`ecflow_client --begin` command to run when you are ready.

## 2.2 What the suite looks like

The C48_ATM suite is a forecast-only case — no data assimilation, no
ensemble. It runs a single GFS forecast cycle at C48 resolution (~200 km),
which completes in minutes on dev hardware. The dependency tree is
straightforward:

```
/C48_ATM_ecflow
  └── gfs/
      └── 2021032312/
          ├── prep → fcst → post → ...
          └── (all tasks in a single linear chain)
```

This makes it useful as a smoke test: if the ecFlow plumbing works
(server communication, `.ecf` rendering, trigger evaluation, Slurm
submission), the forecast will complete.

## 2.3 How the architecture fits together

```
Entry point:      c48_atm_ecflow.py
                       │
Orchestrator:     load_ecflow_case.run()
                       │
                  ┌────┴────┐
                  │         │
Experiment:  setup_expt  setup_workflow ──► ecflow_suite_factory
                              │
Task defs:   ecflow_tasks_factory ──► GFSEcFlowTasks
                              │          (one method per task)
Suite gen:   GFSForecastOnlyEcFlowSuite.write()
                              │
Output:      {pslot}.def + ecf_scripts/
                              │
Server:      ecflow_client --load / --begin
                              │
Execution:   .ecf scripts ──► head.h + slurm.h + J-Job + tail.h
```

The `.ecf` script for each task is a thin wrapper: it includes `head.h`
(which initializes the ecFlow client connection and sets up traps), then
sources `slurm.h` (PBS/Slurm directives), then calls the actual J-Job
script, and finally includes `tail.h` (which calls `ecflow_client --complete`
to mark the task done).

## 2.4 Directory layout after a successful run

```
${RUNTESTS}/
  EXPDIR/C48_ATM_ecflow/           ← experiment config
    config.base                     config files
    config.fcst
    config.atmos_products
    ...
    C48_ATM_ecflow.def              generated ecFlow definition
    ecf_scripts/                    copied .ecf files + manifest
  COMROOT/C48_ATM_ecflow/           ← output data
    gfs.20210323/12/                 forecast output
      model_data/atmos/history/      atmospheric history files
    logs/                            task log files
  RUNDIRS/C48_ATM_ecflow/           ← runtime scratch (cleaned up)
```

## 2.5 Why `--overwrite` exists

The script prompts before deleting previous run directories because
accidental deletion of a multi-hour run is painful. `--overwrite` skips the
prompt and forces cleanup — useful when iterating quickly, but dangerous if
you point it at a run you haven't archived yet.

---

# Section 3 — Troubleshooting

## 3.1 "Missing environment variables"

```
[ERROR] Missing environment variables: ECF_HOST, ECF_PORT, ECF_HOME
```

**Fix:** Source the environment variables from §1.3. Make sure `module load
ecflow` has been run and `unset ECF_HOSTFILE` is set.

## 3.2 "Cannot reach ecFlow server"

```
[ERROR] Cannot reach ecFlow server at <hostname>:<port>
```

**Fix:** Check that the server is running (`ecflow_client --ping`). Verify
the hostname and port match where you started the server. If it's not
running, start it with `ecflow_start.sh` (see §1.4).

## 3.3 Suite delete times out

```
[WARN] Delete failed: timeout
```

**Fix:** The suite has active jobs blocking the delete. Kill them first:
```bash
ecflow_client --suspend /C48_ATM_ecflow
ecflow_client --kill /C48_ATM_ecflow
sleep 20
ecflow_client --delete=force yes /C48_ATM_ecflow
```

## 3.4 "ecflow_client --load" fails

```
[ERROR] ecflow_client failed: ecflow_client --load=...
  stderr: <error message>
```

**Fix:** The `.def` file has a syntax error. Check the generated file:
```bash
cat ${EXPDIR}/C48_ATM_ecflow/C48_ATM_ecflow.def
```

Common causes:
- Task names with special characters
- Missing `endfamily` or `endsuite` closing tags
- Invalid trigger expressions

Validate the `.def` before loading:
```bash
ecflow_client --check ${EXPDIR}/C48_ATM_ecflow/C48_ATM_ecflow.def
```

## 3.5 Tasks stay in "queued" state

**Check triggers:** The task might be waiting for an upstream task.
```bash
ecflow_client --get_state /C48_ATM_ecflow/gfs/2021032312/<task_name>
```

**Check Slurm:** The job might be pending in the queue.
```bash
squeue -u ${USER}
```

## 3.6 Tasks abort immediately

**Check the .ecf script exists:**
```bash
ls ${EXPDIR}/C48_ATM_ecflow/ecf_scripts/<task_name>.ecf
```

**Check the job output:**
```bash
cat ${ECF_HOME}/<task_name>.1
```

Common causes:
- `load_modules.sh` failure (missing modules)
- J-Job script not found (`HOMEglobal` path wrong)
- File permissions

## 3.7 METplus or archive tasks appear when they shouldn't

If `metp` or `arch_tars` show up despite being disabled, check the rendered
`config.base`:
```bash
grep DO_METP ${EXPDIR}/C48_ATM_ecflow/config.base
grep DO_ARCHCOM ${EXPDIR}/C48_ATM_ecflow/config.base
```

On Ursa, these should be `"NO"` by the platform guards in `config.base.j2`.
If they show `"YES"`, the experiment was generated before the guards were
added — regenerate with `--overwrite`.

## 3.8 Quick-reference failure table

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `Missing environment variables` | Env not sourced | Source §1.3 variables |
| `Cannot reach ecFlow server` | Server not running | `ecflow_client --ping`; start if needed |
| `Delete failed: timeout` | Active jobs blocking delete | Suspend → kill → sleep → delete |
| `--load` fails with syntax error | Bad `.def` file | `ecflow_client --check` the def |
| Tasks stuck in queued | Trigger dependency or Slurm queue | Check `--get_state` and `squeue` |
| Tasks abort immediately | Missing `.ecf`, bad paths, permissions | Check script exists; read job output |
| Unwanted metp/archive tasks | Stale experiment config | Regenerate with `--overwrite` |
