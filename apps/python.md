# Python App

*General-purpose execution on Stampede3 — Python by default, any executable via `BINARY`*

| | |
| --- | --- |
| App ID | `python-s3` |
| Platform | DesignSafe / TACC Stampede3 |
| Runtime | ZIP (HPC Batch) |
| Queue (default) | skx |
| Try it | [Run in the workspace](https://www.designsafe-ci.org/workspace/python-s3?appVersion=1.0.0) |

`python-s3` is the recommended general-purpose app for workflows that do not have a dedicated DesignSafe app: post-processing pipelines, parameter sweeps, `mpi4py` codes, machine-learning jobs, OpenSeesPy, and user-compiled solvers. Instead of registering a new app for each analysis type, you bring an input directory and a script; everything else is optional configuration.

Every job runs a staged lifecycle and always writes a machine-readable run record:

```
setup (unzip, modules, pip)  →  PRE_SCRIPT  →  main (BINARY, default python3)  →  POST_SCRIPT
```

- A failing setup or pre-script **aborts before the main run**, so a broken environment never burns SUs.
- A failing main script fails the job and skips the post-script.
- A failing post-script is a warning, unless `POST_SCRIPT_REQUIRED=True`.
- `job-summary.json` is written next to `tapisjob.out` on success *and* failure, with per-stage exit codes and timings, the exact command, the resolved binary, the Python environment, and the loaded modules.

(python-quick-start)=
## Quick start: parallel Python on one node

A minimal but genuinely parallel example — Monte Carlo estimation of π using all 48 cores of a Stampede3 SKX node with `concurrent.futures`, no MPI and no pip installs required.

`pi.py` (place it in a folder in My Data, e.g. `MyData/pi-demo/`):

```python
"""Monte Carlo estimate of pi across all cores of one node."""
import os
import random
import sys
from concurrent.futures import ProcessPoolExecutor


def count_hits(n: int) -> int:
    rng = random.Random(os.getpid())
    return sum(rng.random() ** 2 + rng.random() ** 2 <= 1.0 for _ in range(n))


if __name__ == "__main__":
    samples = int(sys.argv[1]) if len(sys.argv) > 1 else 10_000_000
    workers = len(os.sched_getaffinity(0))
    chunk = samples // workers
    with ProcessPoolExecutor(workers) as pool:
        hits = sum(pool.map(count_hits, [chunk] * workers))
    total = chunk * workers
    print(f"pi ~= {4 * hits / total:.6f}  ({workers} workers, {total:,} samples)")
```

### Submit from the portal

1. Open the [Python app workspace](https://www.designsafe-ci.org/workspace/python-s3?appVersion=1.0.0).
2. **Input Directory**: browse to the folder containing `pi.py`.
3. **Input Script**: `pi.py`.
4. **Arguments** (optional): `50000000` to raise the sample count.
5. Submit. When the job finishes, the archived output in My Data contains `tapisjob.out` (with the π estimate) and `job-summary.json`.

### Submit from Jupyter with dapi

```python
from dapi import DSClient

ds = DSClient()

input_uri = ds.files.to_uri("/MyData/pi-demo")

job_dict = ds.jobs.generate(
    app_id="python-s3",
    input_dir_uri=input_uri,
    script_filename="pi.py",
    max_minutes=10,
    allocation="<your-tacc-allocation>",
    queue="skx-dev",  # development queue: fast turnaround for short runs
    archive_system="designsafe",
    archive_path="python-s3-results",
    job_name="mc-pi",
    description="Monte Carlo pi on one Stampede3 node",
    tags=["demo"],
)

submitted = ds.jobs.submit(job_dict)

# timeout_minutes bounds the monitoring, not the job — it defaults to the
# job's max_minutes, which queue and staging waits can exhaust.
final = submitted.monitor(interval=15, timeout_minutes=60)
ds.jobs.interpret_status(final, submitted.uuid)

for item in ds.files.list(submitted.archive_uri):
    print(item.name)
```

See [Job Resources](../guide/job-resources.md) for choosing node counts, core counts, and walltimes for larger runs.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| Input Directory | yes | Folder containing the main script and all supporting files; the job runs inside it |
| Input Script | yes | Filename (no path) passed to the binary |
| Arguments | no | Free-form command-line arguments appended after the script |

The final command is always `[MPI launcher] BINARY INPUTSCRIPT ARGS...`. Not sure which options your job needs? Work through [Choosing Inputs](python-inputs.md), a one-page decision guide.

## Environment variables

All optional; empty values are skipped entirely.

| Variable | Default | Description |
| --- | --- | --- |
| `BINARY` | `python3` | Executable for the main run: a module-provided name, `./name` in the Input Directory, or an absolute path |
| `USE_MPI` | `False` | Launch the main run with the MPI launcher |
| `MPI_LAUNCHER` | `ibrun` | MPI launch command when `USE_MPI=True` |
| `UNZIP_INPUTS` | (empty) | Comma-separated ZIPs in the Input Directory to expand before anything else runs |
| `EXTRA_MODULES` | (empty) | Comma-separated TACC modules to load for the run |
| `PYTHON_ENV` | (empty) | Path to a persistent virtual environment (created if missing, kept after the job) |
| `PIP_PACKAGES` | (empty) | Comma-separated packages to pip install into the job's Python environment |
| `PIP_REQUIREMENTS` | (empty) | requirements.txt-style file to install; the job fails if the file is missing |
| `PRE_SCRIPT` | (empty) | Script run before the main script; missing or failing aborts the job |
| `POST_SCRIPT` | (empty) | Script run after the main script |
| `POST_SCRIPT_REQUIRED` | `False` | If `True`, a failing post-script fails the job |

## Python environments

When `PIP_PACKAGES` or `PIP_REQUIREMENTS` is set, packages are installed into a **temporary per-job virtual environment** that inherits the module-provided numpy/scipy/mpi4py stack. It is activated for the whole job (pre/post scripts and the main run) and deleted on exit — nothing leaks into `$HOME`, no packages carry between jobs, and the archive stays small.

For repeat jobs, set `PYTHON_ENV` to a persistent path such as `$WORK/envs/myproject`: it is created on first use, reused afterwards (skipping reinstall time), and never deleted. Keep persistent environments in `$WORK` or `$SCRATCH`, not inside the Input Directory.

**Note:** `module load` inside a pre/post script only affects that script. Modules the main run needs must go in `EXTRA_MODULES`.

## Pre- and post-scripts

`PRE_SCRIPT` and `POST_SCRIPT` name a file in the Input Directory (or an absolute path on Stampede3). Files ending in `.py` run with `python3`; executables run directly; anything else runs with `bash`.

Staging and packaging tasks are pre/post-script one-liners rather than app options:

```bash
# pre-script: copy shared inputs from Work
cp -r "$WORK/shared-motions" .
```

```bash
# post-script: package results and move them to Scratch
zip -r -q results.zip output/
mkdir -p "$SCRATCH/runs/$_tapisJobUUID"
mv results.zip "$SCRATCH/runs/$_tapisJobUUID/"
```

Use `POST_SCRIPT_REQUIRED=True` when the post-processing output *is* the deliverable (for example, aggregating a parameter sweep).

(python-bundle-inputs)=
## Bundle many small input files

Tapis stages each file of a directory input as its own transfer task — roughly 40 seconds per file when the tenant transfer queue is busy — so an input directory with dozens of small files can spend longer staging than computing. Ship one ZIP instead and let the app expand it:

```
Input Directory: contains inputs.zip (and nothing else)
UNZIP_INPUTS=inputs
```

The expansion runs before everything else in the job, so the pre-script, `PIP_REQUIREMENTS` file, and main script can all live inside the bundle. One staged file, same job.

## Running other executables

`BINARY` accepts a module-provided name (resolved on PATH after `EXTRA_MODULES` load), `./name` for a program shipped in the Input Directory (the execute bit is restored automatically if staging dropped it), or an absolute path such as `$WORK/apps/mysolver`. Resolution happens after the pre-script — so a pre-script may build the binary — and an unresolvable binary fails the job before the run starts.

OpenSees-MP (Tcl) on 2 nodes, for example:

```
BINARY=OpenSeesMP   EXTRA_MODULES=opensees,hdf5/1.14.4   USE_MPI=True
Input Script: model.tcl
```

With dapi, add the variables to the generated job request before submitting:

```python
job_dict["nodeCount"] = 2
job_dict["coresPerNode"] = 48
job_dict["parameterSet"]["envVariables"] = [
    {"key": "BINARY", "value": "OpenSeesMP"},
    {"key": "EXTRA_MODULES", "value": "opensees,hdf5/1.14.4"},
    {"key": "USE_MPI", "value": "True"},
]
```

(python-openseespy)=
For OpenSeesPy, keep `BINARY=python3`, set `EXTRA_MODULES=opensees,hdf5/1.14.4`, and use a pre-script to copy the TACC-compiled module: `cp ${TACC_OPENSEES_BIN}/OpenSeesPy.so ./opensees.so`, then `import opensees as ops` in your script. Dedicated OpenSees apps are also available — see [OpenSees](opensees.md).

## The run record

Every job writes `job-summary.json`. This one is the actual record from the quick-start job above, fetched from the archive with `submitted.get_output_content("job-summary.json")`:

```json
{
  "app_id": "python-s3",
  "app_version": "1.0.0",
  "job_uuid": "03c94346-56a2-4f6d-9161-edaebdc18a19-007",
  "hostname": "c454-003.stampede3.tacc.utexas.edu",
  "started": "2026-08-07T19:36:03Z",
  "ended": "2026-08-07T19:36:04Z",
  "input_script": "pi.py",
  "binary": "/opt/apps/python/3.12.11/bin/python3",
  "python_env": null,
  "command": "/opt/apps/python/3.12.11/bin/python3 pi.py",
  "python_version": "Python 3.12.11",
  "loaded_modules": "…",
  "stages": {
    "setup": {"seconds": 0},
    "pre_script": {"script": null, "exit_code": null, "seconds": null},
    "main": {"exit_code": 0, "seconds": 1},
    "post_script": {"script": null, "exit_code": null, "seconds": null}
  },
  "exit_code": 0,
  "total_seconds": 1
}
```

The ten million samples took one second across 48 cores (`pi ~= 3.142032`); the three-minute wall time of the job was staging and scheduler overhead, which is why [bundling inputs](#python-bundle-inputs) matters for anything with many files.

`null` means the stage did not run. Together with the archived inputs, this record documents exactly what executed — useful for debugging, provenance, and publication.

## Parameter sweeps

For many small independent runs inside one allocation, combine this app with PyLauncher (the `pylauncher` module is loaded automatically for Python runs). See [Parameter Sweeps](../guide/parameter-sweeps.md).
