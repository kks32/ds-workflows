# Python App: Choosing Inputs

*A one-page cheat sheet for `python-s3`*

The app gives you a lot of options; most jobs need almost none of them. Work through these questions top to bottom, set only what applies, and leave everything else blank — empty values are skipped entirely.

---

## 1. What am I actually running?

Pick one executable and one main script.

| If you are running | Set `BINARY` | Set Input Script |
| --- | --- | --- |
| General Python | (leave unset — `python3` is the default) | `script.py` |
| OpenSeesPy | (leave unset) + see [OpenSeesPy](python.md#python-openseespy) | `run.py` |
| OpenSeesMP / OpenSeesSP (Tcl) | `OpenSeesMP` / `OpenSeesSP` + `EXTRA_MODULES=opensees,hdf5/1.14.4` | `model.tcl` |
| A program shipped in your input directory | `./mysolver` | `input.dat` |
| Your own compiled code on the system | `$WORK/apps/mysolver` | `input.dat` |

Rules of thumb:

- Input Script is a filename, not a path, and must live inside the Input Directory.
- A `BINARY` *name* is resolved on PATH after `EXTRA_MODULES` load; a *path* may be `./relative` (exec bit restored automatically) or absolute.
- The job fails before the run starts if the binary cannot be found.
- For serial OpenSees Tcl, the dedicated [OpenSees apps](opensees.md) are usually simpler.

---

## 2. Do I need MPI?

Answer explicitly; the app will not guess.

| If your job | Set `USE_MPI` |
| --- | --- |
| Uses OpenSeesMP / OpenSeesSP | `True` |
| Uses `mpi4py` | `True` |
| Is fully serial | `False` |
| Uses threading or `concurrent.futures` on one node | `False` |

Mental model:

```text
USE_MPI = True   →  ibrun <command>     (launcher configurable via MPI_LAUNCHER)
USE_MPI = False  →  <command>
```

If unsure, start with `False` and turn it on only when needed. The [quick-start example](python.md#python-quick-start) fills a whole 48-core node with `USE_MPI=False` via `concurrent.futures`.

---

## 3. Does my environment need anything special?

**Modules** — if you would type `module load …` before running on the command line, declare it:

```text
EXTRA_MODULES = opensees,hdf5/1.14.4
```

Modules must go here, not in a pre-script — `module load` inside a pre/post script only affects that script.

**Python packages** — choose one:

| If you | Use |
| --- | --- |
| Have a `requirements.txt` | `PIP_REQUIREMENTS` |
| Just need a couple of packages | `PIP_PACKAGES` |
| Rerun similar jobs and want to skip reinstalling | `PYTHON_ENV=$WORK/envs/myproject` |
| Need none | (leave blank) |

Installs go into a temporary per-job environment that is removed after the run — nothing leaks into `$HOME` or carries between jobs. With `PYTHON_ENV`, the environment persists at the path you name and is reused on the next job.

---

## 4. How are my inputs structured?

**Everything already in the input directory?** Do nothing — simplest and safest.

**Many small files?** Bundle them — Tapis stages each file as a separate transfer (~40 s each under load), so one ZIP stages in seconds where a hundred loose files take an hour:

```text
UNZIP_INPUTS = mydata,meshes        # .zip suffix optional
```

The expansion runs before everything else, so even your pre-script and requirements file can live inside the bundle.

**Large data already on the system?** Copy it in with a pre-script instead of re-uploading:

```bash
# pre-script
cp -r "$WORK/shared-motions" .
```

---

## 5. Do I need pre- or post-processing?

| Need | Input |
| --- | --- |
| Generate or stage inputs | `PRE_SCRIPT` |
| Post-process, organize, or clean up results | `POST_SCRIPT` |

Scripts live in the Input Directory (`.py` runs via python3, anything else via bash) and run inside it. **Failure semantics matter here:** a missing or failing pre-script **aborts the job before the main run** — no SUs are wasted on a broken setup. A failing post-script only warns, unless you set `POST_SCRIPT_REQUIRED=True` because the post-processing output *is* the deliverable.

---

## 6. Will my output be large or long-lived?

Package or relocate results with a post-script:

```bash
# post-script
zip -r -q results.zip output/
mkdir -p "$SCRATCH/runs/$_tapisJobUUID"
mv results.zip "$SCRATCH/runs/$_tapisJobUUID/"
```

Keeping the Tapis archive small speeds up archiving and keeps My Data tidy; `$WORK`/`$SCRATCH` destinations suit chained workflows and interactive inspection from JupyterHub.

---

(python-starter-configs)=
## 7. Minimal starter configurations

### General Python (verified quick-start)

```text
Input Script = pi.py
```

That's the entire configuration of the [worked example](python.md#python-quick-start) — 48 cores, no MPI, no installs.

### OpenSeesMP (MPI)

```text
BINARY        = OpenSeesMP
Input Script  = model.tcl
USE_MPI       = True
EXTRA_MODULES = opensees,hdf5/1.14.4
```

### OpenSeesPy

```text
Input Script  = run.py
EXTRA_MODULES = opensees,hdf5/1.14.4
PRE_SCRIPT    = setup.sh          # copies TACC-compiled OpenSeesPy.so
```

### Python + mpi4py

```text
Input Script = run.py
USE_MPI      = True
PIP_PACKAGES = mpi4py
```

---

## 8. Final sanity checklist

- Is my script inside the Input Directory?
- Does `USE_MPI` match how the code actually runs?
- Are runtime modules in `EXTRA_MODULES` (not in a pre-script)?
- Are ZIP bundles listed in `UNZIP_INPUTS`?
- Should a post-script failure fail the job (`POST_SCRIPT_REQUIRED`)?

After the run, check `job-summary.json` in the archive — it records what actually executed (resolved binary, environment, per-stage exit codes and timings), which answers most "what did my job do?" questions without re-running anything.

---

## 9. Run the demo

The end-to-end example — a pure Python workflow chosen precisely because the app is not tied to any software stack — is worked twice:

- **Portal and dapi walkthrough** on the [Python App page](python.md#python-quick-start), including the real `job-summary.json` from the verified run.
- **Runnable notebook** in the dapi examples: [python-s3-pi.ipynb](https://jupyter.designsafe-ci.org/hub/user-redirect/lab/tree/CommunityData/dapi/python/python-s3-pi.ipynb) with the [step-by-step guide](https://designsafe-ci.github.io/dapi/docs/examples/python).

The core idea carries across every job on DesignSafe: if you can describe your job as

```text
<executable> <script> <arguments>
```

you can run it with this app — the input directory defines the execution context, MPI is explicit, environments are declared rather than assumed, and every run leaves a machine-readable record.
