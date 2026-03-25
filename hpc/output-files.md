# SLURM-Job Output

## Reading `.out` and `.err` Files

When you submit a SLURM job using `sbatch job.slurm`, SLURM automatically captures your program's output in two files.

| File                 | Description                                                                                    |
| -------------------- | ---------------------------------------------------------------------------------------------- |
| *output.<jobid>.out* | Captures standard output, including anything your script prints via `print`, `puts`, stdout, or from the solver     |
| *output.<jobid>.err* | Captures standard error, including runtime errors, syntax issues, missing files, MPI problems, etc. |

These files are automatically named using the job ID and should be your first stop when debugging a failed or stalled job.

## Example

You will find the output files in the base folder of where your SLURM job was executed, or transferred to.

### What to Look For

The `.out` file (standard output) logs the normal progress of your job.

* Echoed input parameters or timestamps
* Printed results or summaries
* Completion messages from OpenSees or your script
* Debugging output you inserted (e.g., `print("step 1 done")`)

If this file is empty, your script may have failed very early, before any standard output was produced.

The `.err` file (standard error) contains critical diagnostics.

* Tcl or Python syntax errors
* Missing or misnamed input files
* Failed module loads (e.g., a missing OpenSees executable)
* MPI startup issues (OpenSeesSP or OpenSeesMP)
* Permission problems (e.g., trying to write to a read-only directory)

Even if your job produced output, always check the `.err` file. It may reveal warnings or silent failures that do not stop the job but indicate something went wrong.

## Best Practice

Always check both files after a job finishes (or fails). These files are stored with your input script. You can view them in JupyterHub or the Data Depot, which you can access from the Job-Status page.

In most cases, these files will tell you exactly what went wrong, or confirm that your job completed successfully.

If you are debugging an OpenSees model, this is where you will find the stack trace of a failed script, errors about file paths, mesh loading, or convergence issues, and output from `puts` or `print` statements that help trace execution.


:::{dropdown} Failed Job Example

Check the files.

The `.err` file in this example shows an MPI error.

```
mpirun: error: unable to open hostfile: No such file or directory
ERROR: MPI process failed to start
child process exited abnormally
```

The `.out` file is empty.

```
<empty>
```

In this case, MPI tried to launch OpenSeesSP but could not find the required hostfile. This typically happens when you did not request multiple nodes but your environment needs one, when your `mpirun` configuration does not match the scheduler, or when OpenSees is not correctly installed or loaded.

:::

## Troubleshooting Checklist

Use this list when your SLURM job does not behave as expected.

|  Step | What to Check                                                    | File            |
| ------ | ---------------------------------------------------------------- | --------------- |
| 1      | Did the job run at all? Look for start/stop messages.            | `.out`          |
| 2      | Is there a syntax or runtime error?                              | `.err`          |
| 3      | Any missing input files or path typos?                           | `.err`          |
| 4      | Are MPI commands/formats correct?                                | `.err`          |
| 5      | Are you using the correct executable (OpenSees, OpenSeesSP)?     | `.out` / `.err` |
| 6      | Does the output stop partway through? Check for crashes.         | `.out` / `.err` |
| 7      | Does your script use absolute or relative paths?                 | Both            |
| 8      | Is there a SLURM-specific error (e.g., exceeded time limit)?     | `.err`          |

```{tip}
An empty `.out` file usually means your job failed before any output was printed. An empty `.err` file is a good sign, but check `.out` for unexpected early exits. If you are still unsure, insert `puts "Starting..."` or `print("Reached step 2")` to help trace the point of failure.
```
