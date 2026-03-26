# Debugging HPC Jobs

This page covers two related topics: understanding parallel execution (because misconfiguring it is a common source of failures) and diagnosing jobs that fail, hang, or produce wrong results.

## Parallel execution and MPI

A single CPU core executes instructions one at a time. For a small model, that is fast enough. But a bridge with millions of finite elements, a coastal surge simulation spanning hundreds of kilometers, or turbulent flow through complex geometry can take days on one core. Parallel computing splits the problem so many cores work on it simultaneously.

[MPI](https://www.mpi-forum.org/) (Message Passing Interface) is how parallel processes exchange data on [TACC](https://www.tacc.utexas.edu/) systems. Each process (called a **rank**) has its own memory and works on a different part of the problem. When one rank needs data from another (for example, forces at a shared boundary between two mesh partitions), MPI handles the exchange.

Several [DesignSafe](https://designsafe-ci.org) applications use MPI internally:

| Application | Parallelism | Typical Use |
|---|---|---|
| [OpenSees](https://opensees.berkeley.edu/) MP | MPI | Structural analysis with domain decomposition |
| [ADCIRC](https://adcirc.org/) | MPI | Coastal and storm surge modeling |
| [OpenFOAM](https://www.openfoam.com/) | MPI | Computational fluid dynamics |
| MPM | MPI | Material point method simulations |
| OpenSees (serial variants) | Serial (no MPI) | Single-process structural analysis on VM or HPC |
| Agnostic App | Serial (no MPI) | General-purpose Python, OpenSeesPy, [PyLauncher](https://github.com/TACC/pylauncher) |
| [MATLAB](https://www.mathworks.com/products/matlab.html) | Serial (no MPI) | Data analysis and algorithms |

Researchers using MPI applications do not write MPI code themselves. The application handles the parallelism internally. The researcher's role is to request the right number of nodes and cores and ensure the model is set up for the intended number of parallel processes.

### Submitting a parallel job

A parallel job looks the same as any other [dapi](https://designsafe-ci.github.io/dapi/) submission, with `node_count` and `cores_per_node` set to match the problem. Here is an OpenSees MP example that runs a structural model across 96 cores (2 nodes, 48 cores each):

```python
from dapi import DSClient

ds = DSClient()
input_uri = ds.files.to_uri("/MyData/opensees/bridge-model/")

job_request = ds.jobs.generate(
    app_id="opensees-mp-s3",
    input_dir_uri=input_uri,
    script_filename="analysis.tcl",
    node_count=2,
    cores_per_node=48,
    max_minutes=120,
    allocation="your_allocation",
    queue="skx",
)

job = ds.jobs.submit(job_request)
job.monitor()
```

This produces 96 MPI ranks (2 x 48). The OpenSees model must be set up for 96 parallel processes. If the number of ranks does not match the model's domain decomposition, the simulation will either fail or produce incorrect results.

| dapi parameter | Effect on MPI |
|---|---|
| `node_count` | Number of physical machines |
| `cores_per_node` | MPI ranks per machine |
| Total ranks | node_count x cores_per_node (must match the model's decomposition) |

### Ranks and how they map to hardware

When an MPI program starts, [SLURM](https://slurm.schedmd.com/documentation.html) launches multiple copies of it. Each copy is a rank with a unique integer ID from 0 through N-1. Requesting 2 nodes with 48 cores each produces 96 ranks. Rank 0 might process the left portion of a structural mesh, rank 47 the right portion on the first node, and ranks 48 through 95 cover the second node.

Every rank executes the same program, but each operates on different data. Ranks cooperate by exchanging messages through the MPI library.

SLURM sets environment variables that a program can read to determine its identity:

| Variable | Meaning |
|---|---|
| `SLURM_PROCID` | Global MPI rank ID (0 through N-1) |
| `SLURM_LOCALID` | Rank index within the current node |
| `SLURM_NODEID` | Node index within the job allocation |
| `SLURM_JOB_ID` | Scheduler job identifier |

### Rank-aware file management

Two ranks writing to the same filename will overwrite each other, producing corrupted output or silently wrong results. Use the rank ID in output file paths to prevent collisions.

```python
import os

rank = os.environ.get("SLURM_PROCID", "0")
output_file = f"results_rank{rank}.txt"
```

All ranks on the same node share the same `/tmp` directory, so even temporary files need rank-based naming.

### Using node-local storage for performance

For I/O-intensive jobs, copying data to `/tmp` on the compute node avoids shared-filesystem contention. Each node has its own `/tmp` partition — it is not shared across nodes and is purged at the end of each job.

| System | Node Type | /tmp Size |
|---|---|---|
| Stampede3 | SKX | 90 GB |
| Stampede3 | ICX | 200 GB |
| Stampede3 | SPR | 150 GB |
| Stampede3 | PVC / H100 | 3.5 TB |
| Frontera | CLX | 144 GB |
| Lonestar6 | All | 288 GB |

```bash
RANK="${SLURM_PROCID:-0}"
SCRATCH_DIR="/tmp/${USER}/job_${SLURM_JOB_ID}/rank_${RANK}"
mkdir -p "${SCRATCH_DIR}"

cp input_${RANK}.dat "${SCRATCH_DIR}/"
cd "${SCRATCH_DIR}"
./solver input_${RANK}.dat > output_${RANK}.dat

# Copy results back before the job ends
cp output_${RANK}.dat "${TAPIS_JOB_WORKDIR}/"
```

Use the shared filesystem (Work/Scratch) for inputs, final outputs, and checkpoints. Use `/tmp` only for high-frequency scratch I/O and intermediate files.

### When parallel computing is not needed

Not every workflow benefits from MPI.

[PyLauncher](https://github.com/TACC/pylauncher) sweeps run many independent serial tasks inside a single SLURM allocation. Each task is a separate process that does not communicate with the others. A fragility study with 500 ground-motion records, each processed by a standalone OpenSeesPy script, is a perfect fit for PyLauncher and has nothing to do with MPI. See [Parameter Sweeps](parameter-sweeps.md).

Single serial scripts that process data sequentially need only one core. Requesting more cores does not speed them up.

Small models that fit on one node and complete in a reasonable time gain nothing from MPI overhead. Running on one core is simpler and avoids the complexity of domain decomposition.

## Job states

[Tapis](https://tapis.readthedocs.io/en/latest/) tracks every job through a sequence of states from submission to completion. Knowing which state a job reached narrows the problem to a specific part of the pipeline.

| State | What It Means |
|---|---|
| PENDING | Submitted but not yet started. |
| STAGING_INPUTS | Tapis is copying input files to the execution system. |
| QUEUED | Waiting in the SLURM scheduler queue for compute resources. |
| RUNNING | Actively executing on compute nodes. |
| ARCHIVING | Tapis is copying output files back to the archive location on Corral. |
| FINISHED | Completed successfully and outputs were archived. |
| FAILED | Something went wrong during staging, execution, or archiving. |
| CANCELLED | Manually cancelled before completion. |

```python
# Check the current state
job.get_status()

# Poll until completion, printing each state transition
job.monitor()
```

### Job stuck in QUEUED

A job that stays in QUEUED for a long time is not broken. It is waiting for SLURM to allocate the requested hardware. Common reasons for long waits:

- The system is busy. Check the [TACC system status page](https://tacc.utexas.edu/portal/system-status/Stampede3) for current load.
- The job requested many nodes or a long walltime. Smaller requests fit into scheduling gaps more easily.
- The queue has a per-user job limit. Check the queue tables in [Running HPC Jobs](job-resources.md).

To reduce wait time, try the development queue (`skx-dev`) for test runs, request fewer nodes, or request a shorter walltime.

## Reading the output files

Every job produces two log files.

`tapisjob.out` is the standard output stream (stdout). Programs print their normal progress messages here, along with results and debugging statements. If a structural analysis prints iteration counts or convergence norms, they appear in this file.

`tapisjob.err` is the standard error stream (stderr). Programs report problems here. Runtime errors, missing file messages, MPI startup failures, and SLURM warnings all appear in this file. When something goes wrong, this is usually the first place to look.

| File | What to Look For |
|---|---|
| `tapisjob.out` | Progress messages, printed results, completion indicators |
| `tapisjob.err` | Syntax errors, missing files, failed module loads, MPI problems, permission errors |

Both files are archived with the job outputs. There are several ways to view them:

- **JupyterHub.** Navigate to the archive directory shown in the job output. The files are plain text and can be opened directly in JupyterHub's file browser or read with Python (`open("tapisjob.err").read()`).
- **Data Depot.** Go to the Job Status page in the DesignSafe portal, find the job, and browse its output files.
- **dapi.** Use `job.list_outputs()` to see archived files, then `job.get_output_content("tapisjob.err")` to read them programmatically.

An empty `.out` file usually means the script failed before producing any output. The error will be in `.err`. Always check `.err` even when the job produced output, because it may reveal warnings or silent failures.

## Troubleshooting checklist

When a job does not behave as expected, work through these checks in order.

| Step | What to Check | Where to Look |
|---|---|---|
| 1 | Did the job run at all? Look for start/stop messages. | `.out` |
| 2 | Is there a syntax or runtime error? | `.err` |
| 3 | Any missing input files or path typos? | `.err` |
| 4 | Are MPI commands and core counts correct? | `.err` |
| 5 | Is the correct executable being used (OpenSees, OpenSeesSP, OpenSeesMP)? | `.out` / `.err` |
| 6 | Does output stop partway through, indicating a crash? | `.out` / `.err` |
| 7 | Does the script use absolute or relative paths correctly? | Both |
| 8 | Is there a SLURM-specific error, such as an exceeded time limit? | `.err` |

## Common failure patterns

**Allocation expired or invalid.** The job fails immediately at the QUEUED stage with a message like `Unable to allocate resources: Invalid account or account/partition combination specified`. Verify the allocation code on the [TACC Dashboard](https://tacc.utexas.edu/portal/dashboard) and confirm it has remaining SUs.

**Input files not found.** Path typos are the most common cause. Tapis stages files into a working directory, so relative paths in the script must match the directory structure that was uploaded. Look in `.err` for messages like `No such file or directory: 'inputs/motions/GM_01.txt'`.

**Walltime exceeded.** SLURM kills jobs that exceed their `max_minutes` limit. The `.err` file will contain `CANCELLED AT ... DUE TO TIME LIMIT`. Resubmit with a longer walltime. A good rule of thumb is to add 50% margin beyond the expected runtime.

**MPI configuration wrong.** On TACC systems, `ibrun` is the correct way to launch MPI applications. If `.err` shows `ORTE was unable to reliably start one or more daemons` or MPI hostfile errors, check that the node/core counts match the model's decomposition.

**Wrong number of MPI ranks.** If an OpenSees MP model is partitioned into 96 subdomains but the job requests 48 cores, the simulation will fail or produce incorrect results. Total ranks (node_count x cores_per_node) must match the model setup.

**Ranks writing to the same file.** If output files contain garbled data or results seem randomly wrong, check that each MPI rank writes to a unique filename. See [Rank-aware file management](#rank-aware-file-management) above.

**Module not available.** If `.err` shows `Lmod has detected the following error: The following module(s) are unknown`, the execution system may not have the expected software version. Verify the app definition includes the correct module loads.

## Reconnecting to a running job

A lost notebook session does not mean losing track of a job. Reconnect using the job UUID.

```python
from dapi import DSClient

ds = DSClient()
job = ds.jobs.get("your-job-uuid-here")
job.get_status()
```

The job UUID appears in the output from `ds.jobs.submit()` and on the DesignSafe Job Status page in the portal.

## Quick debugging workflow

When a job fails, work through these steps:

```python
# 1. Check the final state
job.get_status()

# 2. List the output files
job.list_outputs()

# 3. Read the error log
print(job.get_output_content("tapisjob.err"))

# 4. Read the output log
print(job.get_output_content("tapisjob.out"))
```

Most problems become clear from the error log. If not, work through the [Troubleshooting checklist](#troubleshooting-checklist) above.
