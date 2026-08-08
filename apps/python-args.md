# App Arguments

*Every `python-s3` input and what it does*

The following covers every input available in the portal form and the Tapis job request. Most jobs need only the two required inputs; everything else defaults to off. For a decision-oriented guide, see [Choosing Inputs](python-inputs.md).

## 1. File input

### Input Directory (required)

A single directory staged by [Tapis](https://tapis.io/) into the job working directory.

What should be inside:

- your main script (Python, Tcl, or an input file for your own binary)
- any supporting files your script needs (models, data, configs)
- optional helper files
  - `requirements.txt` (for `PIP_REQUIREMENTS`)
  - pre/post scripts (for `PRE_SCRIPT` / `POST_SCRIPT`)
  - ZIP bundles referenced by `UNZIP_INPUTS`

The wrapper `cd`s into this directory before running anything. Relative paths in your script should assume this directory is the working directory.

---

## 2. App arguments

### Input Script (required)

The filename of the input script passed to the executable.

Rules:

- filename only (no path)
- must exist inside the Input Directory (or inside a bundle expanded by `UNZIP_INPUTS`)

Examples: `run_analysis.py`, `model.tcl`, `input.dat`

### Arguments (optional)

Free-form arguments appended after the Input Script.

Example:

```text
--NodalMass 4.19 --outDir outCase1
```

Final command structure:

```bash
[MPI_LAUNCHER] <BINARY> <Input Script> <Arguments...>
```

---

## 3. Scheduler inputs

### TACC Scheduler Profile (fixed)

The app uses the `python_default` profile, which loads `python/3.12.11` and nothing else. All other module state is controlled explicitly through `EXTRA_MODULES`, so runs are reproducible rather than dependent on login-node defaults.

### Allocation

The DesignSafe portal injects your TACC allocation automatically. From dapi, pass `allocation="..."` to `ds.jobs.generate`.

---

## 4. Environment variables

All optional. Empty values are skipped entirely. "True-like" means `True` or `1` (case-insensitive).

| Variable | Default | One-line summary |
| --- | --- | --- |
| `BINARY` | `python3` | Executable for the main run |
| `USE_MPI` | `False` | Launch through the MPI launcher |
| `MPI_LAUNCHER` | `ibrun` | MPI launch command when `USE_MPI` is true-like |
| `EXTRA_MODULES` | (empty) | TACC modules to load for the run |
| `UNZIP_INPUTS` | (empty) | ZIP bundles to expand before anything else |
| `PYTHON_ENV` | (empty) | Persistent virtual environment to use/create |
| `PIP_PACKAGES` | (empty) | Packages to pip install |
| `PIP_REQUIREMENTS` | (empty) | requirements.txt-style file to pip install |
| `PRE_SCRIPT` | (empty) | Script to run before the main run |
| `POST_SCRIPT` | (empty) | Script to run after the main run |
| `POST_SCRIPT_REQUIRED` | `False` | Whether a failing post-script fails the job |

### 4.1 Execution

#### BINARY (default `python3`)

The executable for the main run. Three forms:

- **Name** (no `/`): resolved on PATH after `EXTRA_MODULES` load — e.g. `OpenSeesMP`
- **`./name`**: a program shipped inside the Input Directory; the execute bit is restored automatically if staging dropped it
- **Absolute path**: e.g. `$WORK/apps/mysolver` for your own compiled code

Resolution happens *after* the pre-script (so a pre-script may build or stage the binary). If the binary cannot be found, the job fails with a clear hint **before the main run starts**. `python` is normalized to `python3`. The resolved absolute path is recorded in `job-summary.json`.

#### USE_MPI (default `False`)

| Value | What runs |
| --- | --- |
| `False` | `<BINARY> <Input Script> [args...]` |
| `True` | `ibrun <BINARY> <Input Script> [args...]` |

Use `True` for OpenSeesMP/SP and `mpi4py`. Use `False` for serial code and single-node parallelism via threading or `concurrent.futures`.

#### MPI_LAUNCHER (default `ibrun`)

The launch command used when `USE_MPI` is true-like. `ibrun` is correct on TACC systems; the option exists so the same wrapper ports to systems using `srun` or `mpirun`.

### 4.2 Modules

#### EXTRA_MODULES

Comma-separated TACC modules to load before anything runs, e.g.

```text
opensees,hdf5/1.14.4
```

A module that fails to load aborts the job. **Modules the main run needs must go here, not in a pre-script** — `module load` inside a pre/post script only affects that script's own process.

### 4.3 Python packages and environments

#### PIP_REQUIREMENTS

A requirements-style file in the Input Directory, e.g. `requirements.txt`. The job **fails immediately** if the file is missing — a silently skipped install is a debugging trap.

#### PIP_PACKAGES

Comma-separated packages, e.g. `scipy,h5py`. Installed in one `pip install` call.

#### PYTHON_ENV

Where installs go depends on this setting:

- **Unset (default):** when either pip input is set, a temporary per-job virtual environment (`.job-venv`) is created with access to the module-provided numpy/scipy/mpi4py stack, activated for the whole job, and **deleted on exit** — nothing leaks into `$HOME`, no packages carry between jobs, the archive stays small. No environment is created when no installs are requested.
- **Set** (e.g. `$WORK/envs/myproject`): that environment is used instead — created if missing, activated, **kept after the job** — so repeat jobs skip reinstall time. Keep persistent environments in `$WORK`/`$SCRATCH`, not inside the Input Directory (they would be archived with your results).

Either way the environment is activated for pre/post scripts and the main run, and its path is recorded as `python_env` in `job-summary.json`. If you override `BINARY` to a *different* Python interpreter by absolute path, it bypasses the environment and will not see the installed packages.

### 4.4 Input preparation

#### UNZIP_INPUTS

Comma-separated ZIP files in the Input Directory to expand **before anything else runs** (`.zip` suffix optional):

```text
UNZIP_INPUTS = mydata,meshes
```

Use this when your input directory would otherwise contain many small files: Tapis stages each file of a directory input as its own transfer task (~40 s per file under tenant load), so one bundled ZIP stages in seconds where a hundred loose files can take an hour. Because expansion runs first, the pre-script, requirements file, and main script can all live inside the bundle. A listed ZIP that is missing fails the job immediately.

### 4.5 Pre/post scripts

#### PRE_SCRIPT

Script run after environment setup but before the main run — use it for staging, generating inputs, or copying data from `$WORK`/`$SCRATCH`.

- if relative, resolved inside the Input Directory
- `.py` runs via `python3`; executables run directly; anything else runs via `bash`
- **a missing or failing pre-script aborts the job before the main run** — a broken setup never burns SUs on the compute phase

#### POST_SCRIPT

Script run after a successful main run (same resolution rules) — use it for post-processing, packaging, or moving outputs. Skipped if the main run fails.

#### POST_SCRIPT_REQUIRED (default `False`)

By default a failing post-script logs a warning and the job still succeeds — a plotting glitch should not mark a ten-hour simulation FAILED. Set `True` when the post-processing output *is* the deliverable (e.g., aggregating a parameter sweep).

---

## 5. Outputs and the run record

Every job writes two log artifacts next to `tapisjob.out`:

- **`job-summary.json`** — machine-readable run record: resolved binary, Python environment, exact command, per-stage exit codes and timings. Written on success *and* failure. See the [example on the app page](python.md#python-quick-start).
- The Tapis archive (in My Data by default) contains the Input Directory, `tapisjob.out`, and `job-summary.json`.

Large outputs are best packaged or relocated by a `POST_SCRIPT` — see [Choosing Inputs](python-inputs.md) for recipes.

---

## 6. Typical patterns

Minimal starter configurations for General Python, OpenSeesMP, OpenSeesPy, and mpi4py are collected in [Choosing Inputs](python-inputs.md#python-starter-configs); the end-to-end verified example is on the [Python App page](python.md#python-quick-start).
