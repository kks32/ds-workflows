# PyLauncher

## Running Parameter Studies with PyLauncher (Basic Usage)

PyLauncher is a lightweight task launcher developed at TACC for running many independent jobs within a single SLURM allocation. It is especially useful for parameter studies, sensitivity analyses, and embarrassingly parallel workloads where each run is independent.

Instead of submitting dozens (or hundreds) of separate SLURM jobs, PyLauncher allows you to submit one SLURM job and execute a list of commands concurrently across the allocated cores. On DesignSafe, PyLauncher integrates cleanly with OpenSeesPy and can be used from the Portal, JupyterHub, or a notebook-driven workflow.

This section documents basic PyLauncher usage only, using the `ClassicLauncher`. MPI-enabled launchers are out of scope here and require additional configuration and testing.


---

## When to Use PyLauncher

Use PyLauncher when you have many independent OpenSeesPy runs, each run is serial (no MPI), each run can write its output to a separate directory, and you want to efficiently use allocated cores inside a single SLURM job.

Do not use PyLauncher if your runs depend on shared output files or you need tight inter-process communication.

If your model requires MPI, use more advanced PyLauncher options, which are beyond the scope of this document.

---

## Basic PyLauncher Workflow


### 1. Create the PyLauncher Input Script

Your main Python script should contain only the PyLauncher invocation.

```python
import pylauncher

pylauncher.ClassicLauncher(
    "runsList.txt",
    debug="host+job"
)
```

The `debug` option is optional and can be removed once things are working. This script does not run OpenSees directly. It only launches the commands listed in `runsList.txt`.

---

### 2. Create the `runsList.txt` File

The `runsList.txt` file contains one command per line, exactly as you would run it from the command line.

```text
python Ex1a.Canti2D.Push.argv.tacc.py --NodalMass 11.79 --outDir outCase39
python Ex1a.Canti2D.Push.argv.tacc.py --NodalMass 11.99 --outDir outCase40
python Ex1a.Canti2D.Push.argv.tacc.py --LcolList 100,200,300 --outDir outCase41
python Ex1a.Canti2D.Push.argv.tacc.py --LcolList 105,205,305 --outDir outCase42
python Ex1a.Canti2D.Push.argv.tacc.py --LcolList 110,210,310 --outDir outCase43
```

Each command must write to its own output directory. Output directories should be specified explicitly (e.g., `--outDir outCaseXX`). Command-line arguments can be passed normally, including lists.

---

### 3. Submit the Job

You may submit the SLURM job in one of two ways. You can write your own SLURM-job file and submit it via `sbatch` from a login node, or you can use a Tapis App, which builds the SLURM-job file and submits the job for you.

#### 3a. When Using a Tapis App

If using OpenSeesPy, use the OpenSeesPy app on DesignSafe. Otherwise use the DesignSafe Agnostic app, which offers many additional features.

Set `UseMPI = false` when submitting the job to the Classic Launcher. You may submit the job through the DesignSafe Portal, from JupyterHub, or from a Jupyter notebook.

Request enough cores to accommodate the number of concurrent tasks you want PyLauncher to run.

---

## Output and Runtime Behavior

Each PyLauncher task runs in its own temporary working directory. Results are copied back to the specified output directories when complete. PyLauncher creates a subdirectory inside your `inputDirectory` that contains per-task stdout, per-task stderr, and launcher bookkeeping files.

Note that the launcher creates many files.

### Logging Behavior

Your script's output does not appear directly in `tapisjob.out`. Instead, logs are written to individual files inside the PyLauncher directory. These files typically do not have extensions.

Inspect PyLauncher output files using JupyterHub, as the Data Depot preview may not interpret these files correctly.

---

## How PyLauncher Fits into a SLURM Job

PyLauncher runs inside a single SLURM allocation. SLURM provides the compute resources, and PyLauncher manages how individual serial tasks are dispatched across those resources.

The flow works as follows.

1. The SLURM job starts and resources (nodes and cores) are allocated by SLURM.

2. PyLauncher is invoked. The `ClassicLauncher` reads a list of commands from `runsList.txt`.

3. Independent tasks are executed. Each command is launched as a separate serial process, using available cores.

4. Temporary execution directories are used. Each task runs in its own isolated working directory.

5. Results are copied back. Output files are written to user-specified directories (e.g., `outCaseXX`).

This model avoids repeated queue waits and allows efficient use of allocated resources for large parameter studies.


---
## Practical Notes and Limitations

* Each task must be file-system isolated
* Shared writes will cause race conditions or data loss
* PyLauncher is ideal for parameter sweeps, not tightly coupled workflows
* MPI-enabled PyLauncher workflows require additional investigation and are not covered here

---

## PyLauncher at TACC

PyLauncher is the supported task launcher at TACC for running multiple serial jobs within a single SLURM allocation.

* The legacy launcher is deprecated and being retired
* PyLauncher is actively supported and maintained
* MPI modes exist but are not documented here
* For serial parameter studies, `ClassicLauncher` is the recommended and stable approach

---

:::{dropdown} Launcher Support Status at TACC

The legacy (old) TACC launcher is no longer supported and is being phased out across all TACC systems. Users should not rely on the old launcher for new or existing workflows.

PyLauncher is the supported replacement for launching multiple tasks within a single SLURM allocation on TACC systems, including those accessed through DesignSafe.

If you encounter older documentation or examples referencing the legacy launcher, those materials should be considered deprecated.

:::


:::{dropdown} MPI Usage and Scope of This Documentation

PyLauncher does support MPI-enabled execution modes, and these can be used to launch MPI tasks under certain configurations.

However, MPI-enabled PyLauncher workflows require additional setup, involve tighter coordination with SLURM and MPI runtimes, and require careful validation and testing.

MPI usage with PyLauncher is intentionally not covered in this documentation.

This section focuses exclusively on the `ClassicLauncher`, serial independent tasks, and reliable and reproducible parameter studies.

If your workflow requires MPI, use a native MPI job configuration or treat PyLauncher's MPI modes as advanced usage outside the scope of this guide.

:::

# Common Mistakes

## Common Mistakes in Using PyLauncher and How to Avoid Them

### Writing all runs to the same output directory

Multiple tasks overwrite files or fail unpredictably when they share a directory.

Ensure every command writes to a unique output folder.

```text
--outDir outCase41
--outDir outCase42
```

---

### Expecting output in tapisjob.out

PyLauncher captures stdout/stderr per task, not globally. You will not see OpenSees or Python output in `tapisjob.out`.

Look inside the PyLauncher-created subdirectory in your `inputDirectory`. Each task has its own log files.

---

### Viewing PyLauncher logs in the Data Depot

PyLauncher log files often have no file extension, so they may appear unreadable or empty in the Data Depot.

Use JupyterHub to inspect PyLauncher output files.

---

### Forgetting to disable MPI

Jobs hang, crash, or behave inconsistently when MPI is enabled for serial tasks.

Set `UseMPI = false` when submitting the job.

```
UseMPI = false
```

---

### Using PyLauncher for tightly coupled workflows

PyLauncher is for independent serial tasks, not coupled simulations. Jobs fail or produce incorrect results when tasks depend on each other.

Use MPI directly (outside PyLauncher) for coupled models.

# Job Arrays vs Launcher

## Comparing SLURM Job Arrays and PyLauncher

On Stampede3, "sarray" refers to Slurm job arrays (`sbatch --array`), and Launcher/PyLauncher refers to TACC's parametric job launcher. Neither is globally "better." Each wins in different regimes.

Below is a comparison, followed by some rules of thumb.

---

## Mental Picture

A Slurm job array is one Slurm submission that creates many independent jobs underneath, each with an index (`SLURM_ARRAY_TASK_ID`). Stampede3 explicitly supports them. ([TACC Documentation][1])

A Launcher/PyLauncher job is one Slurm job allocation inside which TACC's launcher (or PyLauncher) runs a workpile (command list), feeding tasks to allocated cores until the list is exhausted. ([Texas Advanced Computing Center][2])

Historically TACC discouraged arrays on some systems and pushed Launcher/PyLauncher instead, but on Stampede3 both arrays and PyLauncher are available. ([Scribd][3])

---

## Side-by-Side Comparison

:::{dropdown} 1. Queue behavior and time-to-solution


With job arrays, each array element is its own schedulable job (shares an ArrayJobId, but has its own JobId). ([Slurm Documentation][4]) Slurm can start some tasks early via backfill and others later; they do not all need to run at once. The overhead is that the scheduler handles N jobs. If you have thousands of short tasks (say 10-60 seconds), scheduler/launch overhead and file-system churn become noticeable.

With Launcher/PyLauncher, you wait in the queue once for a chunk of nodes. Once you get them, Launcher or PyLauncher keeps them busy with many tasks until the paramlist is done. ([Texas Advanced Computing Center][2]) For tiny tasks (seconds), this usually gives better overall time-to-solution because you amortize scheduler overhead across many tasks, avoid flooding Slurm with thousands of jobs, and can keep nodes saturated even as tasks finish at different times. For long tasks (tens of minutes or more), the advantage shrinks and queue behavior dominates.

As a rule of thumb, tasks of 10-15 minutes or more work well with either arrays or Launcher. Tasks of a few minutes or less, numbering in the thousands, generally run faster with Launcher/PyLauncher.

:::

:::{dropdown} 2. Scheduler and policy friendliness

TACC emphasizes job bundling and minimizing scheduler load. Their job-bundling paper explicitly calls out Launcher/PyLauncher and similar tools for this use case. ([ACM Digital Library][5])

Arrays are more scheduler-intensive when you push to large N (Slurm has a MaxArraySize and sites may set limits). ([Slurm Documentation][4]) A single Launcher/PyLauncher job with a big command list often better aligns with TACC's preferences for high-throughput/ensemble work.

If you are doing Tapis-style thousands of short OpenSees runs or parameter sweeps, Launcher/PyLauncher is more aligned with TACC's "job bundling" philosophy.

:::

:::{dropdown} 3. Implementation complexity

Job arrays are simple to implement. You write one Slurm script with one executable and vary inputs using `$SLURM_ARRAY_TASK_ID`. ([RCC Users][6]) This approach works well when each job follows the pattern of "run this one command with a slightly different input file."

Launcher/PyLauncher requires two pieces. You need the Slurm job script (requesting N nodes, etc.) and a command list/paramlist or a small Python snippet for PyLauncher. ([Texas Advanced Computing Center][2]) There are slightly more moving parts, but one line per task can be arbitrarily complex (different executables, options, working dirs), and PyLauncher has modes for threaded, MPI, and GPU tasks. ([TACC Documentation][7])

If your workload is already "one line per run" (e.g., your OpenSees param list), it maps very naturally to Launcher/PyLauncher.

:::

:::{dropdown} 4. Fault tolerance and restart

With job arrays, each array element is an independent job. If element 172 fails, you can re-submit or rerun just `--array=172` or `172-200`. ([RCC Users][6]) This is great when inputs are noisy and you expect some failures.

With Launcher/PyLauncher, a node failure generally kills the entire Slurm job, though the launcher may have completed many tasks already. PyLauncher has restart/replay support via restart files, but you have to set that up. ([TACC Documentation][7])

For messy workflows with lots of expected per-task failures, arrays can be simpler to reason about. For clean parameter sweeps, launcher works fine.

:::

:::{dropdown} 5. Resource usage and packing

With job arrays, each element requests its own resources (nodes, tasks, memory). Slurm may pack array elements onto nodes efficiently, but from the user's perspective you do not directly control how many array tasks per node beyond the resource requests.

With Launcher/PyLauncher, you explicitly pick nodes and tasks, and Launcher multiplexes your commands onto those cores. This makes it easier to run several light jobs per core or per node (when they are I/O-bound or low CPU) and to tune concurrency (e.g., 8 tasks/node on SKX vs 28 on SPR) to match your application's profile. ([TACC Documentation][1])

:::

---

## Which Should You Use on Stampede3?

### Use Slurm Job Arrays When

* Each task is moderately heavy (10-15 minutes or more)
* Every task looks essentially the same, such as `ibrun ./my_mpi_app ...` or `python script.py param`
* You want native Slurm features (dependencies, array-task-limited runs with `--array=1-1000%32`, etc.)
* You want easy per-task monitoring, accounting, and failure handling

### Use Launcher / PyLauncher When

* You have lots of small or medium tasks, especially 10^2 to 10^5 runs with runtimes from a few seconds to a few minutes
* You want to bundle them into one Slurm job to reduce scheduler and filesystem stress, improve aggregate throughput, and keep a node allocation saturated with work
* You need flexibility, such as heterogeneous commands (different apps/args per line) or multi-threaded/MPI jobs in the same workpile (PyLauncher) ([TACC Documentation][7])

---

## Direct Answers to Three Questions

**Which is "better"?** Policy-wise on Stampede3, for classic HTC/param sweeps, TACC leans toward Launcher/PyLauncher as the recommended tool. Arrays are fine but easier to abuse at scale. ([ACM Digital Library][5])

**Which is "faster"?** For short/high-count tasks, Launcher/PyLauncher generally gives better time-to-solution. For long, heavy tasks, the performance difference is negligible. Pick whichever scripting model suits you.

**Main differences in one sentence.** Job arrays create many independent Slurm jobs with simple `$SLURM_ARRAY_TASK_ID` logic and work well for medium/long embarrassingly parallel runs with clean per-job accounting. Launcher/PyLauncher creates one Slurm job acting as your own mini-scheduler and is ideal for bundling huge ensembles of small/medium jobs while being friendly to the scheduler and file system.



[1]: https://docs.tacc.utexas.edu/hpc/stampede3/ "Stampede3 User Guide - TACC HPC Documentation"
[2]: https://tacc.utexas.edu/research/tacc-research/launcher/ "The Launcher - Texas Advanced Computing Center"
[3]: https://www.scribd.com/document/787452419/EijkhoutHPCtutorials "Eijkhout HPCtutorials | PDF | Computer File"
[4]: https://slurm.schedmd.com/job_array.html "Job Array Support - Slurm Workload Manager - SchedMD"
[5]: https://dl.acm.org/doi/fullHtml/10.1145/3437359.3465569 "Tools and Guidelines for Job Bundling on Modern ..."
[6]: https://users.rcc.uchicago.edu/~tszasz/rccdocs/running-jobs/array/index.html "Job Arrays , Research Computing Center Manual"
[7]: https://docs.tacc.utexas.edu/software/pylauncher/ "PyLauncher at TACC"
