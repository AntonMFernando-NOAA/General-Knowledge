# Chapter 1.2 — Running an ecFlow Suite (and how it differs from Rocoto)

> *Why running a small test suite on WCOSS2 starts with a process that
> just sits there listening on a port — and what every command after
> "module load ecflow" is actually telling that process to do.*

This chapter assumes you have already met **Rocoto**, the workflow manager
used on most NOAA development clusters. If you haven't, you can still read
this; just treat Rocoto as "the simple case" and read straight on. If you
have, the comparison should make ecFlow stop feeling alien.

By the end you should be able to start an ecFlow server on WCOSS2, load a
suite into it, kick it off cycle by cycle, and diagnose the most common
failures — including the half-dozen specific ones I burnt an afternoon
finding when I first did this.

---

## 1. The one big difference

Both Rocoto and ecFlow do the same job: take a bunch of jobs, figure out
the order, submit them to the scheduler, watch them run, retry failures.
But one of them does it without any background process at all. The other
won't lift a finger unless you start a process first.

```
Rocoto                                ecFlow
------                                ------
[ workflow.xml ]                      [ ecflow_server (always running) ]
       |                                          |
       v                                          | TCP port (e.g. 2137)
[ rocotorun  ]   <-- cron                +--------+--------+
       |        every 5 min              v                 v
[ SQLite DB  ]                  [ ecflow_client ]  [ ecflow_ui ]
       |                                  |                 |
       v                                  v                 v
[ qsub jobs  ]                       [ qsub jobs ]   (you click things)
```

**Rocoto is cron + a database.** It wakes up every five minutes, reads an
XML file, reads a SQLite database, asks the scheduler what's running,
submits whatever's now eligible, writes state back, and exits. Total
"engine" lifetime: a few seconds, every five minutes.

**ecFlow is a database server + a GUI.** A long-running daemon called
`ecflow_server` holds the entire workflow state in RAM and listens on a
network port. Anything that wants to ask "what's running?" or say "this
job just finished" connects to that port over TCP. Engine lifetime:
forever, until you kill it.

Why the difference? ecFlow was built by ECMWF for ops centers where
operators stare at a colored tree of tasks 24/7 and need sub-second
responses when they click something. Cron-polling can't deliver that.
Rocoto was built for research groups iterating on workflows without
24/7 monitoring. Different needs, different design.

---

## 2. Apartment analogy refresher

If Chapter 1.1 made the "port = apartment number" picture stick, ecFlow
fits right into it. The server lives in *one* apartment of *one* computer.
Every client knocks on that exact door.

```
   ┌─────────────────────────────────────────────┐
   │                  dlogin01                   │
   │                                             │
   │   ...   Apt 22   Apt 80   ...   Apt 2137    │
   │         SSH      Web            ecflow_     │
   │                                 server      │
   └─────────────────────────────────────────────┘
        client points at:  dlogin01 : 2137
                              host    port
```

Everything you'll do is some variation of "send a message to apartment 2137
on dlogin01." That's why the recurring incantation is:

```bash
export ECF_HOST=dlogin01
export ECF_PORT=2137
ecflow_client --ping
```

If you forget either of those, the client falls back to looking at a
system-wide hostfile and ends up knocking on a wrong door (often a
production-ops door it doesn't have keys for), and you get cryptic
"Connection refused on port 34637" errors.

---

## 3. Vocabulary, side by side

| Idea                   | Rocoto                                   | ecFlow                                       |
|------------------------|------------------------------------------|----------------------------------------------|
| The whole workflow     | XML file                                 | suite (defined in a `.def` file)             |
| Grouping of tasks      | metatask                                 | family                                       |
| Unit of work           | task                                     | task                                         |
| Submitted script       | `<command>` in XML                       | a `.ecf` file under `ECF_FILES`              |
| Time-stepped runs      | `<cycledef>`                             | repeat clauses, or one family per cycle      |
| Dependency             | `<taskdep .../>`                         | `trigger ../foo == complete`                 |
| State persistence      | SQLite file                              | server checkpoint files in `ECF_HOME`        |
| What kicks the engine  | cron running `rocotorun`                 | the server is always running                 |
| How you check status   | `rocotostat`                             | `ecflow_client --get_state` or `ecflow_ui`   |
| How you fix a stuck task | `rocotorewind` then `rocotoboot`       | `ecflow_client --requeue=force`              |

A few of these deserve more than a row in a table.

### `.ecf` scripts vs Rocoto's `<command>`

In Rocoto, the XML embeds the actual job command. In ecFlow, the `.def`
just *names* a task; the script lives in a separate `.ecf` file at a
predictable path. When the server decides a task is ready, it:

1. Finds the `.ecf` file by combining `ECF_FILES` and the task's path in
   the suite tree (e.g. `${ECF_FILES}/gfs/forecast/jgfs_fcst.ecf`).
2. Reads it line by line, expanding any `%VARIABLE%` placeholders.
3. Drops the result into a real shell script under `${ECF_HOME}` named
   `<task>.job0`.
4. Runs `${ECF_JOB_CMD}` (which on WCOSS2 is `qsub`) on that file.

That `.job0` file is what you'd `cat` if you wanted to see exactly what
PBS got asked to run.

### Triggers are state, not files

```
trigger ../analysis/jgfs_atmos_anal == complete
```

That does **not** mean "wait until some output file appears." It means
"wait until the ecFlow server's view of that other task says it's
complete." The state lives entirely in the server's memory. So if you
`--requeue` a task, every dependent task that already ran also goes back
to queued, because the server now treats it as "not complete anymore."

### `%VARIABLE%` substitution

ecFlow has its own templating language. When the server renders a `.ecf`
script, any `%FOO%` it finds gets replaced with the value of `FOO` set on
the task, or if not, on the parent family, then the suite, then the
server's default list. If `FOO` is not set anywhere, the substitution
**fails** and the task aborts before ever running. That's the most common
"why did my freshly-loaded suite immediately abort everything?" cause.

---

## 4. The full procedure on WCOSS2

This is the script I wish I'd had on my first attempt. Each step is
labeled with what could plausibly go wrong and how to fix it.

### Step 0 — One-time setup: make a personal env file

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

In every new SecureCRT tab, run:

```bash
source ~/ecflow_c96.env
```

The `unset ECF_HOSTFILE` matters more than you'd think. WCOSS2 ships a
system-wide `/lfs/h1/ops/prod/config/dhostfile` that points at production
servers. If `ECF_PORT` is empty, ecflow_client falls back to that file and
silently aims at the wrong host. You'll see "Failed to connect to
localhost:34637" or "Connection refused on port 34637" with no clue why.
The unset short-circuits that fallback.

### Step 1 — Start the server

`ssh` to `dlogin01` and stay there. The server is bound to whatever node
you start it from. SSH'ing to a different login node mid-session breaks
`localhost` resolution and confuses the client.

```bash
ssh dlogin01
source ~/ecflow_c96.env
mkdir -p "${ECF_HOME}"
ecflow_start.sh -p "${ECF_PORT}" -d "${ECF_HOME}"
ecflow_client --ping
```

You should see:

```
ping server(dlogin01:2137) succeeded in ...
```

That confirms the daemon is up and listening. The server is **persistent**:
it survives logout. Next time you log in, just `--ping` and continue.

#### What if SecureCRT lands me on a different login node?

It will. WCOSS2's generic hostnames (`cactus.wcoss2.ncep.noaa.gov`,
`dogwood.wcoss2.ncep.noaa.gov`) round-robin to whatever login node has
capacity — `dlogin01`, `dlogin02`, etc. So the next time you connect
you may land on `dlogin02` instead of `dlogin01`. **Your server is
still on `dlogin01`**, the node it was started from. Don't restart it,
don't move it; just *aim the client at it from wherever you are*.

All login nodes can talk to each other on the internal network, so
from any new shell:

```bash
source ~/ecflow_c96.env       # ECF_HOST is hard-coded to dlogin01
ecflow_client --ping
```

should still print `ping server(dlogin01:2137) succeeded`. The fact that
you happen to be sitting on `dlogin02` is irrelevant -- `ECF_HOST` says
`dlogin01` and TCP knows how to find it.

Two things to watch for when you cross nodes:

1. **Don't trust `--ping` without explicit flags after a fresh shell.**
   If anything in the shell startup re-points the client at the production
   `dhostfile`, the bare `--ping` lands on the wrong port. The defensive
   form is unambiguous and works from anywhere:

   ```bash
   ecflow_client --host=dlogin01 --port=2137 --ping
   ```

   Use `--host` and `--port` on every command if you want zero ambiguity
   about which server you're talking to. The whole step-by-step procedure
   above does this for safety.
2. **The server will outlive your shell.** Running `ssh dlogin01` from
   another node connects you to the same machine the server is on, but
   killing your terminal session does not kill the server. `ecflow_start.sh`
   detaches the server from your shell with `nohup`. To actually stop it
   you have to send `--terminate=yes` explicitly.

If you ever genuinely want to start over on a fresh login node (because
the original is being drained, or you forgot which one you used), do the
moving deliberately:

```bash
ssh dlogin01
ecflow_client --terminate=yes        # stop the old server
ssh dlogin02
source ~/ecflow_c96.env              # then edit ECF_HOST=dlogin02 first
ecflow_start.sh -p "${ECF_PORT}" -d "${ECF_HOME}"
```

And update `~/ecflow_c96.env` to point at the new node so you don't keep
fighting the old hostname.

### Step 2 — Load the suite, suspended

```bash
cd "${HOMEgfs}"
ecflow_client --delete=force=yes /gfs_c96 2> /dev/null
ecflow_client --load=dev/ecf/c96/defs/gfs_c96.def
ecflow_client --suspend=/gfs_c96
ecflow_client --suites
```

`--load` parses the `.def` file and registers the suite in the server's
memory. **No jobs are submitted yet.** `--suspend` is a safety belt that
prevents anything from auto-submitting even after the begin step.

The last command should print `gfs_c96`. That confirms the load.

If `--load` fails with "Expression node tree references failed", your
`.def` file has a structural bug — typically a trigger pointing at a task
that doesn't exist. Fix the def, requeue the load.

### Step 3 — Set the suite-level variables

This is the step that bit me hardest. The production GFSv17 def assumes a
bunch of variables are set by NCO's machinery on the real ops servers.
On a personal dev server, they're empty, and any task that references
one of them aborts before it even runs, with a message like:

```
EcfFile::variableSubstitution: failed: '%ecflow_ver%'
```

Each missing variable produces an identical-looking error, so you peel
onions. Save yourself the time and set the lot up front:

```bash
ecflow_client --alter add variable HOMEgfs        "${HOMEgfs}"      /gfs_c96
ecflow_client --alter add variable ECF_LOGHOST    "${ECF_HOST}"     /gfs_c96
ecflow_client --alter add variable ecflow_ver     5.6.0             /gfs_c96
ecflow_client --alter add variable PDY            "$(date +%Y%m%d)" /gfs_c96
ecflow_client --alter add variable PARATEST       NO                /gfs_c96
ecflow_client --alter add variable COMPATH        ""                /gfs_c96
ecflow_client --alter add variable MAILTO         ""                /gfs_c96
ecflow_client --alter add variable DBNLOG         NO                /gfs_c96
ecflow_client --alter add variable SENDDBN        NO                /gfs_c96
ecflow_client --alter add variable SENDDBN_NTC    NO                /gfs_c96
ecflow_client --alter add variable SENDCANNEDDBN  NO                /gfs_c96
ecflow_client --alter add variable rrfs_ver       ""                /gfs_c96
```

#### Critical: tell ecFlow to submit through PBS, not run inline

On a freshly-started server, `ECF_JOB_CMD` defaults to running the rendered
job *directly* in the foreground (`%ECF_JOB% 1> %ECF_JOBOUT% 2>&1`). On a
WCOSS2 login node that means your jobs run **on the login node itself**,
which is forbidden — the system reaper kills them after ~5 minutes (the
task aborts with `exit status 124`) and you'll get a polite-but-firm
warning from admins.

Override the three job-handling commands so ecFlow submits to PBS and
talks to PBS for kill / status:

```bash
ecflow_client --alter add variable ECF_JOB_CMD    "qsub %ECF_JOB% 1> %ECF_JOBOUT% 2>&1" /gfs_c96
ecflow_client --alter add variable ECF_KILL_CMD   "qdel %ECF_RID%"                      /gfs_c96
ecflow_client --alter add variable ECF_STATUS_CMD "qstat %ECF_RID% > %ECF_JOB%.stat 2>&1" /gfs_c96
```

Confirm by `--alter`-ing then watching `qstat -u $USER` after `--begin`:
real PBS jobs should appear, with `S=R` (running) on a compute node, not
as a process under your login-node UID.

#### Critical: use a *dev-local* `head.h` and `tail.h`

The shared `ecf/include/head.h` and `tail.h` in this repo are written
for production. Inside `head.h` they call `module load prod_envir`,
which on WCOSS2 silently re-points `ECF_HOST` and `ECF_PORT` at NCO's
operational ecflow servers via `/lfs/h1/ops/prod/config/dhostfile`.

Symptom: jobs reach the compute node, run cleanly, then `ecflow_client
--init` (called from `head.h`) tries to talk to `cdecflow01:34637`
and friends, walks the entire prod-server list, times out (`status 124`),
and from ecFlow's perspective the task stays `submitted` forever even
though PBS finished it.

Fix: keep a dev-local copy of the includes that re-pin `ECF_HOST` /
`ECF_PORT` after the module loads, and point `ECF_INCLUDE` at that
directory instead of the shared one.  In the C96 suite this lives at
`dev/ecf/c96/include/`. The relevant patch in both `head.h` and `tail.h`
is just a few lines, applied right before the `ecflow_client --init` /
`--complete` call:

```bash
# C96 dev override: prod_envir loader silently overrides ECF_HOST/ECF_PORT
# via /lfs/h1/ops/prod/config/dhostfile.  Re-pin them before the call.
unset ECF_HOSTFILE
export ECF_HOST=%ECF_LOGHOST%
export ECF_PORT=%ECF_PORT%
```

If you see the abort log filling with `trying next host(cdecflow01:34637)
... ddecflow02:34637 ...`, this is the issue. The suite-level
`unset ECF_HOSTFILE` you ran in the *driver* shell does not propagate
to the PBS job; the job is a fresh shell on a compute node.

#### A subtle one: `ECF_FILES` and `ECF_INCLUDE` should be absolute

The suite header sets these to `%HOMEgfs%/...`. In theory ecFlow expands
`%HOMEgfs%` at job-render time and you get a real path. In practice, on a
WCOSS2 5.6.0 server, the substitution sometimes misfires — perhaps because
the server reads these *before* the suite-level `HOMEgfs` is added. The
symptom is an error like:

```
Directory ECF_FILES(%HOMEgfs%/dev/ecf/c96/scripts) does not exist
```

If you see that, pin them to absolute paths:

```bash
ecflow_client --alter change variable ECF_INCLUDE "${HOMEgfs}/dev/ecf/c96/include" /gfs_c96/primary
ecflow_client --alter change variable ECF_FILES   "${HOMEgfs}/dev/ecf/c96/scripts" /gfs_c96/primary
```

The `change` (vs `add`) matters: the variable already exists at this scope,
you're rewriting it. `add` would error.

Confirm the result with `--query`:

```bash
ecflow_client --query variable /gfs_c96/primary:ECF_INCLUDE
```

If the printed value still starts with a `/` followed by `dev/ecf/...`
(e.g. `/dev/ecf/c96/include`), the substitution leaked: `%HOMEgfs%`
was empty when ecFlow expanded the variable. That's still wrong even
though it looks structurally correct -- the suite will fail to find
the include files. Re-run the `change` command with the absolute
path explicit.

### Step 4 — Begin only the first cycle

If the suite has multiple cycles (here, 12Z cold-start then 00Z), only let
the first one run while you're watching. Keep the rest suspended.

```bash
ecflow_client --suspend=/gfs_c96/primary/00
ecflow_client --resume=/gfs_c96
ecflow_client --resume=/gfs_c96/primary/12
ecflow_client --begin=gfs_c96
```

`--begin` flips the suite from "loaded" to "running." With 00Z still
suspended, only 12Z's tasks actually go anywhere.

### Step 5 — Watch state

```bash
ecflow_client --get_state /gfs_c96/primary/12 \
  | grep -oE "state:[a-z]+" | sort | uniq -c
```

Healthy progression looks like this, run repeatedly over time:

```
            794 state:queued
   1 state:active   793 state:queued
   3 state:active   1 state:complete   791 state:queued
   ...
   794 state:complete
```

For PBS-side visibility:

```bash
qstat -u "${USER}"
```

### Step 6 — Diagnose aborts

```bash
ecflow_client --get_state /gfs_c96/primary/12 \
  | grep -E "task .* state:aborted" | head -3
```

Each line carries the abort reason after `abort<:`. The common patterns:

| Error contains                                       | Cause                                        | Fix                                            |
|------------------------------------------------------|----------------------------------------------|------------------------------------------------|
| `Directory ECF_FILES(%HOMEgfs%/...) does not exist`  | Variable substitution didn't expand          | Pin `ECF_FILES` absolute (Step 3 last block)   |
| `Could not open include file: .../head.h`            | Same, but for `ECF_INCLUDE`                  | Pin `ECF_INCLUDE` absolute                     |
| `EcfFile::variableSubstitution: failed: '%X%'`       | Variable `X` not set anywhere on suite       | `--alter add variable X <value> /gfs_c96`      |
| Task ran, then aborted partway                       | Real runtime failure (missing data, etc.)    | Read `${ECF_HOME}/<task path>.1`               |
| `exited with status 124` after ~5 min                 | Job ran inline on the login node and was reaped | Set `ECF_JOB_CMD="qsub ..."` (Step 3 sub-block) |
| `trying next host(cdecflow01:34637) ...`              | `prod_envir` overrode `ECF_HOST`/`ECF_PORT` to ops servers via dhostfile | Use a dev-local `head.h`/`tail.h` that re-pins them; `ECF_INCLUDE` -> `dev/ecf/c96/include` |

After fixing a structural cause, requeue:

```bash
ecflow_client --requeue=force /gfs_c96
```

Requeue resets the affected tasks back to "queued." The server then
re-evaluates triggers and re-submits whatever's eligible.

### Reloading after a def or include change

Editing the `.def` file or the include scripts on disk does **not**
automatically update the running server. The server is operating off
an in-memory copy of whatever `--load` last sent it. So after a `git
pull` (or any local edit you want the server to see) you have to
reload, *and re-add every `--alter` variable*, because deleting the
suite drops them with it:

```bash
# 1. Replace the in-memory suite
ecflow_client --delete=force=yes /gfs_c96
ecflow_client --load=dev/ecf/c96/defs/gfs_c96.def
ecflow_client --suspend=/gfs_c96

# 2. Re-add the suite-level vars from Step 3 (every one of them)
#    -- including ECF_JOB_CMD/ECF_KILL_CMD/ECF_STATUS_CMD

# 3. Re-pin ECF_INCLUDE and ECF_FILES absolute (Step 3 last block)

# 4. Resume + begin the cycles you want to run
```

It's easy to forget step 2.  The symptom is exactly the same set of
`failed: '%X%'` aborts you saw on first load.  When in doubt, save
the whole sequence as a small reload script and run it after every
`--delete`.

If you only edited an `.ecf` script, you don't need to reload: the
next time the server renders that task (next submit, or a `--requeue`)
it re-reads the file from disk.

### Step 7 — Release the next cycle

When 12Z is fully green:

```bash
ecflow_client --get_state /gfs_c96/primary/12 \
  | grep -oE "state:[a-z]+" | sort | uniq -c
# expect:  N state:complete
ecflow_client --resume=/gfs_c96/primary/00
```

If your suite has a `cycle_end` task that handles the handoff, that's what
will fire 00Z. Watch the same way.

### Step 8 — Clean up

```bash
ecflow_client --delete=force=yes /gfs_c96
```

That removes the suite from the server's memory; the `.def` and `.ecf`
files on disk are untouched.

To kill the server entirely (only when you're done for the day or week):

```bash
ecflow_client --terminate=yes
```

---

## 5. Why each thing matters

| Ingredient                     | Without it                                                    | With it                                              |
|--------------------------------|---------------------------------------------------------------|------------------------------------------------------|
| `ecflow_server`                | Nothing remembers state; clients can't connect.               | Persistent in-memory state, port to talk to it.      |
| `ECF_HOST` + `ECF_PORT`        | Client uses fallback hostfile, hits production by accident.   | Client reaches your server.                          |
| `ECF_HOME`                     | Server has nowhere to write checkpoints or rendered jobs.     | Crash-resistant; renders jobs you can inspect.       |
| `ECF_FILES`                    | Server can't find the `.ecf` script for any task.             | Tasks render and submit.                             |
| `ECF_INCLUDE`                  | `%include <head.h>` can't be resolved.                        | Standard headers/footers work.                       |
| Suite-level vars in Step 3     | Every task aborts with `failed: '%X%'`.                       | Substitution succeeds, jobs reach PBS.               |
| `--suspend` before `--begin`   | Suite starts running while you're still configuring.          | You stay in control until you say "go."              |
| Staying on the same login node | Mid-session host change orphans the server reference.         | Client and server agree on `localhost`.              |

---

## 6. Common pitfalls

- **"Connection refused on port 34637."** That number is not yours. It
  comes from the production `ECF_HOSTFILE` your env didn't override.
  Fix: `unset ECF_HOSTFILE` and set `ECF_HOST` + `ECF_PORT` explicitly.
- **"My suite loads, then everything goes red instantly."** Variable
  substitution is failing. Look for `EcfFile::variableSubstitution: failed`
  in the abort reason and add the missing variable on the suite.
- **"Files on disk look right but ecflow says the directory doesn't
  exist."** `ECF_FILES` literally contains `%HOMEgfs%/...` because the
  substitution didn't run. Pin to an absolute path with `--alter change`.
- **"I edited the def but the server still has the old one."** Loading
  doesn't re-read the file. You have to `--delete` and `--load` again.
  Variable overrides set by `--alter` get wiped along with the suite, so
  re-apply them.
- **"I `ssh`'d to a different login node and now nothing works."** Your
  server lives on the original node. Either `ssh` back, or set `ECF_HOST`
  to that original node so the new shell aims correctly.
- **"My job aborted with `exited with status 124` after a few minutes."**
  ecFlow ran the job *on the login node* instead of submitting it to PBS,
  and the system reaped it. Override `ECF_JOB_CMD`, `ECF_KILL_CMD`, and
  `ECF_STATUS_CMD` per Step 3 so jobs go through `qsub`. (Running real
  jobs on a login node will also earn you an admin warning.)
- **"PBS finished my job but ecFlow says it's still `submitted`."** The
  job's `--init` callback never reached your dev server. Look at the PBS
  `.o<jobid>` file for `trying next host(cdecflow01:34637)`-style lines.
  Cause: `prod_envir` re-aimed the client at the ops servers. Fix: use a
  dev-local `head.h`/`tail.h` that re-exports `ECF_HOST=%ECF_LOGHOST%` and
  `ECF_PORT=%ECF_PORT%` after the module loads.
- **"`ECF_INCLUDE` looks like `/dev/ecf/c96/include`."** That's a
  half-substituted path: `%HOMEgfs%` expanded to empty.  The suite will
  not find `head.h`. Fix it with `--alter change variable ECF_INCLUDE
  /full/absolute/path /gfs_c96/primary`.
- **"qsub fails with `Error: Please include a valid walltime`."** That
  particular `.ecf` script is missing `#PBS -l walltime=...` (and probably
  the rest of the PBS preamble). Every `.ecf` that goes to `qsub` needs the
  full PBS header at the top, including `select`, `walltime`, `queue`, and
  account. Compare against a working script in the same suite if unsure.
- **"`ecflow_ui` won't open: Qt platform plugin xcb failed."** That's an
  X11 forwarding problem on your *terminal*, not an ecFlow problem. You
  don't need the GUI to run a suite — every check in this chapter uses
  `ecflow_client` from the command line. Solve the GUI later if you want
  to monitor.

---

## 7. The same workflow in Rocoto, briefly

For contrast, here's how the same loop looks on Hera with Rocoto:

```bash
# Generate / edit workflow.xml
vi workflow.xml

# Crontab runs this every 5 min, but you can also run it manually:
rocotorun -d gfs.db -w workflow.xml

# Check status
rocotostat -d gfs.db -w workflow.xml

# A task aborted; reset and retry it
rocotorewind -d gfs.db -w workflow.xml -t MyTask -c 202606040000
rocotoboot   -d gfs.db -w workflow.xml -t MyTask -c 202606040000
```

No daemon. No port. State is in `gfs.db`, a SQLite file. There is no
"loaded into a server" step — Rocoto re-derives state every time it runs.
You can `vi workflow.xml`, save, and the next `rocotorun` picks it up.

It's hard to overstate how much lighter that loop is. The price is: no
real-time GUI, no push notifications when a task finishes, and no shared
view across multiple operators. Tradeoffs.

---

## 8. Mental model to lock in

- **The server is the workflow.** Without it running, ecFlow does nothing.
  Rocoto has no equivalent — its "server" is cron.
- **Clients talk to the server over a port.** Get the host:port wrong and
  you connect to nothing or, worse, the wrong server.
- **State lives in RAM.** A `--load` puts the suite into the server's
  memory; `--alter` mutates that memory; `--delete` evicts it. The `.def`
  file on disk is just the seed.
- **Triggers are about state, not files.** They reference other tasks'
  ecFlow state, not files in the COM area.
- **`%VARIABLE%` is the templating glue.** Most "everything aborted at
  once" failures come down to one missing variable.
- **Suspend before you begin.** Keep the safety belt on while you finish
  setting things up.
- **The job lives on a compute node, not your login shell.** Anything
  it depends on (env vars, paths, hostnames) must be set inside the
  `.ecf` script or its includes -- the driver shell's exports do not
  travel with it.
- **`.ecf` script + variable expansion = the actual PBS job.** Read the
  rendered `.job0` file under `${ECF_HOME}` to see exactly what was
  submitted.

> **One-line shorthand:** *Rocoto is cron + a database. ecFlow is a
> database server + a GUI.*

Once that lands, every command in this chapter is just "ask the running
database server something" or "tell the running database server to change
state." The rest is plumbing.
