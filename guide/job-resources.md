# Running HPC Jobs

This page walks through submitting a simulation to [TACC](https://www.tacc.utexas.edu/) supercomputers through [DesignSafe](https://designsafe-ci.org): how jobs flow through the system, how to submit one, and how to choose the right resources. For background on HPC systems, nodes, queues, and allocations, see [Compute Environments](compute-environments.md).

## Two ways to submit a job

DesignSafe provides two paths to submit an HPC job. Both produce the same result.

**The web portal.** From the [DesignSafe workspace](https://www.designsafe-ci.org/rw/workspace/), select an application (such as OpenSees or OpenFOAM), fill in a form with input files and resource settings, and click Submit. This works well for one-off runs and getting started.

**dapi from a Jupyter notebook.** [dapi](https://designsafe-ci.github.io/dapi/) is a Python library that submits jobs programmatically. Instead of clicking through a form, a few lines of code describe the job, submit it, monitor its progress, and retrieve results. This is the preferred approach when running many jobs, building reproducible workflows, or automating parameter studies.

Both methods use [Tapis](https://tapis.readthedocs.io/en/latest/) behind the scenes. Tapis is the middleware that connects the DesignSafe interface to TACC hardware. It copies input files to the execution system, generates a SLURM scheduler script, submits it, monitors the job, and copies results back. The researcher never writes SLURM scripts or transfers files manually.

## How a job moves through the system

<img src="../images/ComputeWorkflow.jpg" alt="DesignSafe HPC computational workflow" width="75%" />

Every HPC job follows the same sequence, regardless of how it was submitted.

1. **Submit.** The researcher describes the job (application, input files, resources) and submits it through the portal or dapi.
2. **Stage inputs.** Tapis copies input files from DesignSafe storage to the execution system.
3. **Generate and queue.** Tapis creates a SLURM batch script and submits it. The job enters the queue.
4. **Execute.** When the requested nodes become available, SLURM launches the application. The simulation runs and writes output to a working directory on the compute system.
5. **Archive.** After completion, Tapis copies results back to DesignSafe storage.
6. **Retrieve.** The researcher accesses results from JupyterHub, the portal, or dapi.

The wait between steps 3 and 4 is queue time. Development queues typically start within minutes. A large production job may wait longer depending on system load.

## End-to-end example with dapi

This example submits an OpenSees parallel analysis to Stampede3, monitors it, and retrieves the results. It runs from a Jupyter notebook on DesignSafe.

### Step 1: Set up and authenticate

```python
from dapi import DSClient

ds = DSClient()
```

`DSClient()` creates an authenticated connection to DesignSafe services. On JupyterHub, authentication happens automatically.

### Step 2: Point to your input files

```python
input_uri = ds.files.to_uri("/MyData/opensees/site-response/")
```

This converts a DesignSafe path (as seen in the Data Depot) into a Tapis URI that the job submission system requires. The directory should contain the OpenSees script and any supporting files (ground motions, material definitions, etc.).

### Step 3: Generate the job request

```python
import json

job_request = ds.jobs.generate(
    app_id="opensees-mp-s3",
    input_dir_uri=input_uri,
    script_filename="Main.tcl",
    node_count=1,
    cores_per_node=16,
    max_minutes=30,
    allocation="your_allocation",
    queue="skx",
)

print(json.dumps(job_request, indent=2, default=str))
```

`ds.jobs.generate()` builds a complete Tapis job request using the app's defaults, then overrides the fields specified above. Printing the result lets you verify the configuration before submitting.

| Parameter | What it means |
|---|---|
| `app_id` | Which application to run (`opensees-mp-s3` is OpenSees-MP on Stampede3) |
| `input_dir_uri` | Where the input files are stored on DesignSafe |
| `script_filename` | The main script file in the input directory |
| `node_count` | Number of physical machines to allocate |
| `cores_per_node` | CPU cores per node (total cores = node_count x cores_per_node) |
| `max_minutes` | Wall-clock time limit. SLURM kills jobs that exceed it. |
| `allocation` | The TACC project account charged for the job |
| `queue` | Which queue to submit to (see [Compute Environments](compute-environments.md#slurm-and-queues)) |

### Step 4: Submit and monitor

```python
job = ds.jobs.submit(job_request)
print(f"Job UUID: {job.uuid}")

job.monitor(interval=15)
```

`ds.jobs.submit()` sends the request to Tapis and returns a job object. `job.monitor()` polls the job status every 15 seconds and prints each state transition (PENDING, STAGING_INPUTS, QUEUED, RUNNING, ARCHIVING, FINISHED) until the job completes. The job UUID is a unique identifier that can be used to reconnect later.

### Step 5: List and retrieve results

```python
job.list_outputs()
```

After the job finishes, Tapis archives output files back to DesignSafe storage. `list_outputs()` shows what was produced. Results can be accessed directly from JupyterHub at the archive path, or downloaded with `job.get_output_content()`.

## What dapi generates behind the scenes

The `ds.jobs.generate()` call above produces a Tapis job request. Tapis converts it into a SLURM batch script equivalent to:

```bash
#!/bin/bash
#SBATCH -A your_allocation
#SBATCH -p skx
#SBATCH -N 1
#SBATCH -n 16
#SBATCH -t 00:30:00
# ... module loads, file staging, execution commands ...
```

There is no need to write this script by hand. Tapis generates it automatically from the job request parameters.

## Login nodes vs compute nodes

TACC systems have two types of machines. **Login nodes** are shared entry points where researchers land after connecting via SSH or JupyterHub. They are for editing files, submitting jobs, and light scripting. Running a simulation directly on a login node slows the machine for everyone and may be killed automatically.

**Compute nodes** are the machines that actually run jobs. They are accessed only through SLURM. When a job is submitted, SLURM assigns it to one or more compute nodes, and the simulation runs there in isolation. The job's environment (modules, environment variables, working directory) is set up on the compute node, not the login node. This distinction matters when debugging: a command that works in a JupyterHub terminal (login node) may fail inside a job (compute node) if the required modules or paths are different.

## Why Work and Scratch are faster than MyData

MyData and MyProjects live on Corral, a networked storage system optimized for reliability and backup. Work and Scratch live on [Lustre](https://www.lustre.org/), a parallel filesystem designed for high-throughput I/O on HPC clusters. Lustre stripes files across many disks simultaneously, so large reads and writes are significantly faster.

For production simulations, always stage data to Work or Scratch rather than running directly against MyData. The performance difference is especially noticeable for jobs that read or write many files, or that perform frequent I/O during execution.

## Resource sizing guidance

For storage paths, file staging, and transfer tips, see [Storage and File Management](storage.md). For node types, queue specifications, and allocation billing, see [Compute Environments](compute-environments.md).

**Start small.** Run the model in the development queue with a short walltime and a single node. A 10-minute test on `skx-dev` costs almost nothing and catches most configuration errors before they waste hours of allocation time.

**Account for staging time.** Tapis copies input files before the job starts and archives outputs afterward. Both count against the walltime limit. If a simulation takes 30 minutes but staging takes 10, request at least 50 minutes.

**Match cores to the problem.** For [MPI](https://www.mpi-forum.org/) jobs, total ranks = node_count x cores_per_node. If a model is decomposed into 96 subdomains, request 2 nodes with 48 cores each. For serial jobs or [PyLauncher](https://github.com/TACC/pylauncher) sweeps, one node is usually enough.

**Watch memory per core.** All cores on a node share the same RAM. If each MPI process needs 8 GB and the node has 192 GB, running all 48 cores gives only about 4 GB per process. Requesting fewer cores per node gives each process more memory. See the [memory-per-core table](compute-environments.md#nodes-cores-and-memory) for specifics.
