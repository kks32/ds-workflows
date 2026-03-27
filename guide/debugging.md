# Debugging Failed Jobs

A job finished with status FAILED, has been stuck in QUEUED for hours, or completed but the results look wrong. This page walks through how to diagnose and fix these problems.

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

A job that stays in QUEUED for a long time is not broken. It is waiting for [SLURM](https://slurm.schedmd.com/documentation.html) to allocate the requested hardware. Common reasons for long waits:

- The system is busy. Check the [TACC system status page](https://tacc.utexas.edu/portal/system-status/Stampede3) for current load.
- The job requested many nodes or a long walltime. Smaller requests fit into scheduling gaps more easily.
- The queue has a per-user job limit. Check the queue tables in [Compute Environments](compute-environments.md#slurm-and-queues).

To reduce wait time, try the development queue (`skx-dev`) for test runs, request fewer nodes, or request a shorter walltime.

## Reading the output files

Every job produces two log files.

`tapisjob.out` is the standard output stream (stdout). Programs print their normal progress messages here, along with results and debugging statements. If a structural analysis prints iteration counts or convergence norms, they appear in this file.

`tapisjob.err` is the standard error stream (stderr). Programs report problems here. Runtime errors, missing file messages, [MPI](https://www.mpi-forum.org/) startup failures, and SLURM warnings all appear in this file. When something goes wrong, this is usually the first place to look.

| File | What to Look For |
|---|---|
| `tapisjob.out` | Progress messages, printed results, completion indicators |
| `tapisjob.err` | Syntax errors, missing files, failed module loads, MPI problems, permission errors |

Both files are archived with the job outputs. There are several ways to view them:

- **JupyterHub.** Navigate to the archive directory shown in the job output. The files are plain text and can be opened directly in JupyterHub's file browser or read with Python (`open("tapisjob.err").read()`).
- **Data Depot.** Go to the Job Status page in the [DesignSafe](https://designsafe-ci.org) portal, find the job, and browse its output files.
- **dapi.** Use `job.list_outputs()` to see archived files, then `job.get_output_content("tapisjob.err")` to read them programmatically.

An empty `.out` file usually means the script failed before producing any output. The error will be in `.err`. Always check `.err` even when the job produced output, because it may reveal warnings or silent failures.

## Troubleshooting checklist

When a job does not behave as expected, work through these checks in order.

| Step | What to Check | Where to Look |
|---|---|---|
| 1 | Did the job run at all? Look for start/stop messages. | `.out` |
| 2 | Is there a syntax or runtime error? | `.err` |
| 3 | Any missing input files or path typos? | `.err` |
| 4 | Are MPI core counts correct? (see [Parallel jobs](job-resources.md#serial-vs-parallel-jobs)) | `.err` |
| 5 | Is the correct executable being used (OpenSees, OpenSeesSP, OpenSeesMP)? | `.out` / `.err` |
| 6 | Does output stop partway through, indicating a crash? | `.out` / `.err` |
| 7 | Does the script use absolute or relative paths correctly? | Both |
| 8 | Is there a SLURM-specific error, such as an exceeded time limit? | `.err` |

## Common failure patterns

**Allocation expired or invalid.** The job fails immediately at the QUEUED stage with a message like `Unable to allocate resources: Invalid account or account/partition combination specified`. Verify the allocation code on the [TACC Dashboard](https://tacc.utexas.edu/portal/dashboard) and confirm it has remaining SUs.

**Input files not found.** Path typos are the most common cause. Tapis stages files into a working directory, so relative paths in the script must match the directory structure that was uploaded. Look in `.err` for messages like `No such file or directory: 'inputs/motions/GM_01.txt'`.

**Walltime exceeded.** SLURM kills jobs that exceed their `max_minutes` limit. The `.err` file will contain `CANCELLED AT ... DUE TO TIME LIMIT`. Resubmit with a longer walltime. A good rule of thumb is to add 50% margin beyond the expected runtime.

**MPI configuration wrong.** On [TACC](https://www.tacc.utexas.edu/) systems, `ibrun` is the correct MPI launcher (see [Running HPC Jobs](job-resources.md#modules-and-ibrun)). If `.err` shows `ORTE was unable to reliably start one or more daemons` or MPI hostfile errors, check that the node/core counts match the model's decomposition.

**Wrong number of MPI ranks.** If an OpenSees MP model is partitioned into 96 subdomains but the job requests 48 cores, the simulation will fail or produce incorrect results. Total ranks (node_count x cores_per_node) must match the model setup.

**Ranks writing to the same file.** If output files contain garbled data or results seem randomly wrong, check that each MPI rank writes to a unique filename. See [Rank-aware file management](job-resources.md#rank-aware-file-management).

**Module not available.** TACC uses environment **modules** to manage software versions. If `.err` shows `Lmod has detected the following error: The following module(s) are unknown`, the compute node does not have the expected software. This often happens when a module name or version has changed. Verify the app definition includes the correct module loads.

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
