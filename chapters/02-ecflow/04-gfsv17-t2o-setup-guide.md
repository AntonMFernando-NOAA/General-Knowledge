# GFSv17 T2O Setup Guide

> **Status**: Work in progress
> **Last updated**: 7/15/2026

This chapter walks through the full process of cloning, building, configuring, and running the GFSv17 suite on WCOSS2 using ecFlow in NCO mode.

---

## 1. Clone and Build

```bash
mkdir /lfs/h2/emc/${your_group}/noscrub/$USER/gfs/git/release
cd /lfs/h2/emc/${your_group}/noscrub/$USER/gfs/git/release

# Clone the repository
git clone -b dev/gfs.v17 --recursive https://github.com/noaa-emc/global-workflow.git gfs.v17.0.0

# Set gfs.v17.0.0 as HOMEgfs and cd into it
export HOMEgfs=$PWD/gfs.v17.0.0
cd ${HOMEgfs}

# Build all executables and link them
pushd ${HOMEgfs}/sorc
./build_all.sh gfs gdas gsi
./link_workflow.sh -o
popd

# Setup gfs for NCO
pushd ${HOMEgfs}/dev/ush
source gw_setup.sh
./setup_gfs_for_nco.py
popd

# Create ecf links to create files for each forecast hour
pushd ${HOMEgfs}/ecf
./setup_ecf_links.sh
popd
```

---

## 2. Configure the ECF Suite Def File

Edit `${HOMEgfs}/ecf/defs/gfs_prod.def` and change the following variables at the top:

| Variable | Description |
|----------|-------------|
| `PACKAGEHOME` | Location where you cloned global-workflow |
| `ENVIR` | Switch from `prod` to `para` as appropriate |
| `QUEUE` | A queue you can use (default: `dev`; options: `devmax`, etc.) |
| `OUTPUTDIR` | Your output location |
| `EXPDIR` | Full path to `packagehome/parm/config/gfs` |

### Additional variables needed for development

| Variable | Description |
|----------|-------------|
| `DATAROOT` | Path to desired rundir |
| `DBNLOG` | Boolean `YES`/`NO` — whether to log DBN calls |
| `ECF_INCLUDE` | Path to `packagehome/ecf/include` |
| `SENDDBN` | Set to `YES` to test DBN (writes fake alerts to `$OUTPUTDIR/gfs/$RUN.PDY/$cyc/<dbn_alert_name>.log`) |
| `SDATE` | Required — forecast job fails without it |
| `COMROOT` | Perhaps just needs `COMROOT=ROTDIR` |

### Notes

- The `ECF_INCLUDE` change is needed to overwrite `COMROOT` and `DATAROOT` for dev purposes.
- Set `SENDDBN` to `NO` until we sort out whether it's needed and what changes are necessary.
- You can change the suite name — this is **required** if you have multiple suites running.

### Temporary workaround

Until [issue #4972](https://github.com/NOAA-EMC/global-workflow/issues/4972) is resolved, in `config.base` change:

```bash
RUN_ENVIR="emc"
```

---

## 3. Configure `versions/run.ver`

Edit `$HOMEgfs/versions/run.ver` and change:

```bash
export COMPATH="/lfs/h1/ops/prod/com/obsproc:/lfs/h2/emc/ptmp/${USER}/gfs/ops/para/com/gfs"
```

The last part of the path is the per-path to the COM directory and needs to be changed.

- `/lfs/h2/emc/ptmp/${USER}` can be your own path, but `/ops/para/com/gfs` must all be present.
- Your `COMDIR` will then be: `/lfs/h2/emc/ptmp/${USER}/gfs/ops/para/com/gfs/v17.0`
- Or: `(your-custom-path)/ops/para/com/gfs/v17.0`

---

## 4. Get Email Notifications

Update the email list and uncomment the relevant line in `ecf/include/head.h`.

---

## 5. Stage Input Files

> **NOTE**: Need to figure out what to do for machine switch. Tar files are only available for half-cycle starts, not full-cycle starts.

In your `COMDIR`:

```bash
mkdir -p /lfs/h2/emc/ptmp/${USER}/gfs/ops/para/com/gfs/v17.0
cd /lfs/h2/emc/ptmp/${USER}/gfs/ops/para/com/gfs/v17.0

# PDY and CC should be for the PREVIOUS cycle
PDY=YYYYMMDD
CC=HH

mkdir enkfgdas.${PDY}
mkdir gdas.${PDY}

cd enkfgdas.${PDY}
ln -sf /lfs/h2/emc/gfstemp/emc.global/comroot/retrov17_01_realtime/enkfgdas.${PDY}/${CC} ${CC}

cd ../gdas.${PDY}
ln -sf /lfs/h2/emc/gfstemp/emc.global/comroot/retrov17_01_realtime/gdas.${PDY}/${CC} ${CC}

# Copy in the operational syndat files (if they aren't already present)
cd ..
cp -rf /lfs/h1/ops/prod/com/gfs/v16.3/syndat .
```

---

## 6. Run the Suite with ecFlow

> This assumes you already have an ecflow server running. If not, see [Section 11: Set Up ecFlow Server](#11-set-up-ecflow-server-from-scratch).
> If you are running this from a group account, do this from the group account.

### Set the ecFlow host

```bash
# On Cactus (where the group account ecflow server is running)
export ECF_HOST="cdecflow02"

# On Dogwood (where the group account ecflow server is running)
export ECF_HOST="ddecflow02"
```

### If this is the first time (or the server needs a restart)

```bash
ssh ${ECF_HOST}
module load ecflow

# For personal use
export ECF_ROOT=${HOME}/ecflow

# For role account
export ECF_ROOT=/lfs/h2/emc/global/noscrub/emc.global/ecflow

# For all cases
export ECF_OUTPUTDIR=${ECF_ROOT}/output
export LFS_OUTPUTDIR=${ECF_ROOT}/submit
export ECF_COMDIR=${ECF_ROOT}/com
mkdir -p ${ECF_ROOT}/output
mkdir -p ${ECF_ROOT}/submit
mkdir -p ${ECF_ROOT}/com

server_check.sh ${ECF_ROOT}

# NOTE the port number
# If you lose it, it's the same as `id -u` + 2000
# WARNING: do NOT use ecflow_start.sh — it will start on the wrong port!
ecflow_client --restart
exit  # Go back to the login node
```

### Load and begin the suite

```bash
export ECF_PORT=<your_port_number>
module load ecflow
cd $HOMEgfs/ecf/defs/

# The following two lines are done once via the group account
ecflow_client --load $PWD/gfs_prod.def
ecflow_client --begin gfs_v17_nco
```

### Open the ecFlow UI

From your personal account (not the group account):

```bash
ssh -Y ddecflow02
module load ecflow
ecflow_ui &
```

> This is easier than command line at first, but to iterate it is better to submit via commands like:
> ```bash
> ecflow_client --run /gfs_v17_nco/primary/00/gfs/v17.0/gdas/prep/marine/jgdas_marine_obs_dump
> ```

---

## 7. Useful ecFlow Commands

### Delete and reload after changes to the def file

```bash
ecflow_client --delete=/gfs_v17_realtime_test01
ecflow_client --load gfs_prod.def
ecflow_client --begin gfs_v17_realtime_test01
```

### Swap an ecFlow def file

```bash
ecflow_client --replace=/yoursuite1 $PWD/yoursuite2.def
```

> Use `--replace` to swap a suite if already loaded; use `--load` if not loaded yet.

---

## 8. Trigger Jobs to Start a Cycle

There are both atm and marine analysis jobs that depend on the previous cycle — these will have to be manually triggered the first time.

### The "complete previous cycle" trick

You can force-complete the previous cycle so you don't need to manually trigger analysis and enkf jobs. Note that `ECF_PORT` and `ECF_HOST` must be set:

```bash
ecflow_client --force=complete recursive /gfs_v17_nco/primary/HH
```

Where `HH` is the **previous** cycle.

### Jobs to manually trigger (when starting from ICs from rocoto)

#### GFS

- All prep jobs
- If **not** using the ecflow_client trick:
  - `analysis/atmos/jgfs_atmos_anal` — only after atm prep has finished
  - `analysis/marine/jgfs_marine_bmat_init`

#### GDAS

- All prep jobs
- If **not** using the ecflow_client trick:
  - `analysis/atmos/jgdas_atmos_anal` — only after atm prep has finished
  - `analysis/atmos/jgdas_atmos_anal_calc` — after `analysis/atmos/jgdas_atmos_sfc_regrid` and `analysis/atmos/jgdas_atmos_sfc_gcycle`
  - `analysis/marine/jgdas_marine_bmat_init`
  - `analysis/snow/jdas_atmos_analsnow`

#### EnKF

- If **not** using the ecflow_client trick:
  - `enkf/analysis/create/jenkfgdas_atmos_ens_observer` — after gdas atmos prep
  - `enkf/analysis/create/jenkfgdas_snow_anal_ens`

---

## 9. Set Up the Auxiliary Workflow

### Modify `dev/parm/aux/aux.yaml`

| Variable | Description |
|----------|-------------|
| `start_date` | Format `YYYYMMDDHH00` (TODO: should default to `SDATE` in `EXP_aux/config.base`) |
| `end_date` | Some future date (can be short or same as `start_date` for testing) |
| `HOMEglobal` | Only uncomment/set if setup needs to point externally |
| `EXP_aux` | The expdir for the case (TODO: could default to `$HOMEglobal/parm/config/gfs`) |
| `ECF_OUT_gfs` | Where ecflow scripts are written, including `/today` (e.g. `/lfs/h2/emc/gfstemp/emc.global/ecflow/output/today`) |
| `COM_aux` | Same COM as for the GFS (TODO: could be read from `${EXP_aux}/config.base`) |
| `DATAROOT_aux` | DATAROOT for the auxiliary system (TODO: could be read from `$HOMEglobal/ecf/defs/gfs_prod.def`) |
| `output` | Where to render the XML to (TODO: could be `$EXP_aux/aux.xml`) |

### Set variables in `config.base` (if not already done)

| Variable | Value |
|----------|-------|
| `PSLOT` | Something unique like `ecflow_test_XX` |
| `ARCDIR` | Local archiving directory |
| `ATARDIR` | HPSS archiving directory (keep `$PSLOT` in the path) |
| `DO_ARCHCOM` | `YES` |
| `ARCHCOM_TO` | `hpss` |
| `SDATE` | `YYYYMMDDHH` — the actual start date for the ecflow run |

### Run the setup

```bash
dev/ush/gw_setup.sh
cd dev/workflow
./setup_aux.py --config ../parm/aux/aux.yaml
```

The `aux.xml` file should now be in `$HOMEglobal/parm/config/gfs/aux.xml`. Run it via rocoto in cron.

Logs will be written to `$COM/logs/YYYYMMDDHH/<job_name>.log`. Jobs are named the same as rocoto experiments.

---

## 10. Set Up the Cleanup Cron

To clean up the ecflow workflow, set up a cron job to launch `dev/ush/gfsv17_ecflow_rt_cleanup.sh`.

1. Modify the script to point to the ecflow `COMROOT` and `DATAROOT` locations.
2. Add this line to the `emc.global` crontab:

```cron
0 22 * * * /lfs/h2/emc/gfstemp/emc.global/ecflow/packages/gfs.v17.0.0/dev/ush/gfsv17_ecflow_rt_cleanup.sh >& /lfs/h2/emc/gfstemp/emc.global/ecflow/packages/gfs.v17.0.0/dev/ush/gfsv17_ecflow_rt_cleanup.$$
```

---

## 11. Set Up ecFlow Server (from scratch)

Reference:
- [NOAA-EMC/WAFS ecf dev docs](https://github.com/NOAA-EMC/WAFS/tree/develop/dev/ecf)
- [ecFlow setup Google Doc](https://docs.google.com/document/d/1Yoc9AnXEhNHiHkE9i5PwyuDk8ye5kgpIWIMk6aeBfxU/edit?tab=t.0#heading=h.y9m6eb75yk03)

---

## Open Questions

- How to change `dev` → `devmax` — where is queue set? Same as rocoto or somewhere else?
