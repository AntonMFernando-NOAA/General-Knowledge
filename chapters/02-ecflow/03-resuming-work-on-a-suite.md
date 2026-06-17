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

# Section 5 — TL;DR cheat sheet

Pin this to your wall:

```bash
# Every morning:
ssh -Y dlogin01
source ~/ecflow_c96.env
ecflow_client --ping
ecflow_client --get_state | grep server_state         # must say RUNNING
ecflow_client --get_state | grep -E "^suite "         # what's loaded?

# If server is dead:
mkdir -p "${ECF_HOME}"
ecflow_start.sh -p "${ECF_PORT}" -d "${ECF_HOME}"

# If server is SHUTDOWN/HALTED:
ecflow_client --restart

# Clean reload of a suite:
cd "${HOMEgfs}"
ecflow_client --delete=force=yes /gfs_c96 2>/dev/null
ecflow_client --load=dev/ecf/c96/defs/gfs_c96.def
ecflow_client --suspend=/gfs_c96
bash dev/ecf/c96/bootstrap.sh
ecflow_client --resume=/gfs_c96
ecflow_client --begin=gfs_c96

# Open the GUI (if X11 is working):
ecflow_ui &
```
