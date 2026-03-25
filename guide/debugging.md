# Debugging Failed Jobs

A job finished with status FAILED. Or it has been stuck in QUEUED for hours. Or it completed but the results look wrong. The first step is always the same. Check the job state to find where things went off track, then read the log files for the actual error message.

## Checking Job State

[Tapis](https://tapis.readthedocs.io/en/latest/) tracks every job through a sequence of states from submission to completion. Knowing which state a job reached narrows the problem to a specific part of the pipeline.

| State | What It Means |
|---|---|
| PENDING | Submitted but not yet started. |
| STAGING_INPUTS | Tapis is copying input files to the execution system. |
| QUEUED | Waiting in the [SLURM](https://slurm.schedmd.com/documentation.html) scheduler queue for compute resources. |
| RUNNING | Actively executing on compute nodes. |
| ARCHIVING | Tapis is copying output files back to the archive location on Corral. |
| FINISHED | Completed successfully and outputs were archived. |
| FAILED | Something went wrong during staging, execution, or archiving. |
| CANCELLED | Manually cancelled before completion. |

The [dapi](https://designsafe-ci.github.io/dapi/) library provides two ways to check state.

```python
# Get the current state
job.get_status()

# Poll until completion, printing each state transition
job.monitor()
```

The `monitor()` method polls the Tapis API at intervals and prints each state change until the job reaches a terminal state (FINISHED, FAILED, or CANCELLED).

## Reading the Output Files

Every job produces two log files that contain the information needed to diagnose problems.

`tapisjob.out` is the standard output stream (stdout). Programs print their normal progress messages here, along with results and any debugging statements added to the code (such as `print("step 1 done")`). If a structural analysis prints iteration counts or convergence norms, they appear in this file.

`tapisjob.err` is the standard error stream (stderr). Programs report problems here. Runtime errors, missing file messages, [MPI](https://www.mpi-forum.org/) startup failures, and SLURM warnings all appear in this file. When something goes wrong, this is usually the first place to look.

| File | What to Look For |
|---|---|
| `tapisjob.out` | Progress messages, printed results, completion indicators |
| `tapisjob.err` | Syntax errors, missing files, failed module loads, MPI problems, permission errors |

Both files are archived with the job outputs. They can be viewed in JupyterHub or the Data Depot from the Job Status page.

An empty `.out` file usually means the script failed before producing any output. The error that caused the early failure will appear in `.err`. Always check `.err` even when the job produced output, because it may reveal warnings or silent failures that did not stop execution.

## Troubleshooting Checklist

When a job does not behave as expected, work through these checks in order.

| Step | What to Check | Where to Look |
|---|---|---|
| 1 | Did the job run at all? Look for start/stop messages. | `.out` |
| 2 | Is there a syntax or runtime error? | `.err` |
| 3 | Any missing input files or path typos? | `.err` |
| 4 | Are MPI commands and formats correct? | `.err` |
| 5 | Is the correct executable being used (OpenSees, OpenSeesSP, OpenSeesMP)? | `.out` / `.err` |
| 6 | Does output stop partway through, indicating a crash? | `.out` / `.err` |
| 7 | Does the script use absolute or relative paths correctly? | Both |
| 8 | Is there a SLURM-specific error, such as an exceeded time limit? | `.err` |

## Common Failure Patterns

Allocation expired or invalid. The job fails immediately at the QUEUED stage with a message like `Unable to allocate resources: Invalid account or account/partition combination specified`. Verify the allocation code (for example, `TG-ENG230001`) on the [TACC](https://www.tacc.utexas.edu/) Dashboard and confirm it has remaining SUs.

Input files not found. Path typos are the most common cause. Tapis stages files into a working directory, so relative paths in the script must match the directory structure that was uploaded. Look in `.err` for messages like `No such file or directory: 'inputs/motions/GM_01.txt'`. Double-check the spelling and case of every path component.

Walltime exceeded. SLURM kills jobs that exceed their `max_minutes` limit. The `.err` file will contain a message such as `CANCELLED AT ... DUE TO TIME LIMIT`. Resubmit with a longer walltime, or optimize the model to run faster. A good rule of thumb is to add 50% margin beyond the expected runtime for the first few submissions.

MPI configuration wrong. On TACC systems, `ibrun` is the correct way to launch MPI applications. Using `mpirun` or `mpiexec` directly can produce unexpected behavior. If `.err` shows messages like `ORTE was unable to reliably start one or more daemons` or MPI hostfile errors, check that the app definition and node/core counts are correct.

Module not available. If `.err` shows `Lmod has detected the following error: The following module(s) are unknown`, the execution system may not have the expected software version. Verify the app definition includes the correct module loads.

## Reconnecting to a Running Job

A lost notebook session or a switch to a different machine does not mean losing track of a job. Reconnect using the job UUID.

```python
from dapi import SubmittedJob

job = SubmittedJob(ds._tapis, "your-job-uuid-here")
job.get_status()
```

The job UUID appears in the output from the original submission and on the [DesignSafe](https://designsafe-ci.org) Job Status page.
