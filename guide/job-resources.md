# Running HPC Jobs

This page walks through everything involved in running a simulation on one of TACC's supercomputers through DesignSafe: what HPC is, how jobs flow through the system, how to submit one, and how to choose the right resources.

## What is HPC?

A high-performance computing (HPC) system is a cluster of many powerful computers (called **nodes**) connected by a fast network. Each node has dozens of CPU cores and hundreds of gigabytes of memory. By spreading work across many nodes, problems that would take days on a laptop can finish in minutes.

[TACC](https://www.tacc.utexas.edu/) operates several HPC systems that DesignSafe researchers can use: [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/), [Frontera](https://docs.tacc.utexas.edu/hpc/frontera/), and [Lonestar6](https://docs.tacc.utexas.edu/hpc/lonestar6/).

These are shared systems. Thousands of researchers submit jobs to the same hardware, so a program called [SLURM](https://slurm.schedmd.com/documentation.html) (Simple Linux Utility for Resource Management) manages access. SLURM is a **job scheduler**. When a job is submitted, SLURM puts it in a queue. When the requested nodes become available, SLURM allocates them, launches the application, enforces the time limit, and tracks how many resources were used. Researchers using DesignSafe never interact with SLURM directly. Tapis generates SLURM scripts and submits them automatically. But understanding that a queue exists explains why jobs do not start immediately and why requesting the right amount of resources matters.

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
| `queue` | Which queue to submit to (see [Queues](#queues) below) |

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

## Storage paths and file staging

Input files, simulation outputs, and shared datasets move between storage areas during a workflow. The same storage area appears at different paths depending on the environment.

### JupyterHub paths

| Storage Area | Path |
|---|---|
| MyData | `/home/jupyter/MyData/` |
| MyProjects | `/home/jupyter/MyProjects/PRJ-...` |
| Work | `/home/jupyter/Work/stampede3/` |
| CommunityData | `/home/jupyter/CommunityData/` |
| NHERI Published | `/home/jupyter/NHERI-Published/PRJ-...` |

### Stampede3 paths

| Storage Area | Path |
|---|---|
| Home | `/home1/<groupid>/<username>/` |
| Work | `/work2/<groupid>/<username>/stampede3/` |
| Scratch | `/scratch/<groupid>/<username>/` |

Paths on Stampede3 can be verified with `echo $HOME`, `echo $WORK`, and `echo $SCRATCH`.

### dapi path translation

```python
ds.files.to_uri("/MyData/analysis/")
```

This converts a DesignSafe path into a Tapis URI. See the [dapi documentation](https://designsafe-ci.github.io/dapi/) for the full file-management API.

### File transfer tips

Many small files transfer slowly. A directory with 1,000 small CSV files takes significantly longer to stage than a single tar.gz archive. Bundle input files before staging whenever possible.

If multiple jobs share the same input data (for example, 500 ground-motion records reused across a fragility study), keep that data in Work to avoid re-staging it for every submission.

Large datasets benefit from the higher I/O bandwidth of Work and Scratch. Running jobs directly against MyData (Corral) is slower and not recommended for production simulations.

## Stampede3 node types

Stampede3 is the primary execution system for DesignSafe HPC jobs. It contains several node types with different capabilities.

| Node Type | Cores per Node | Memory | Count | Notes |
|---|---|---|---|---|
| ICX (Ice Lake) | 80 | 256 GB DDR4 | 224 | Standard compute nodes |
| SPR (Sapphire Rapids) | 112 | 128 GB HBM2e | 560 | High memory bandwidth, good for memory-bound applications |
| SKX (Skylake) | 48 | 192 GB DDR4 | 1,060 | Older generation, most numerous |
| PVC (Ponte Vecchio) | 96 | 512 GB DDR5 | 20 | Intel GPUs for ML and GPU-accelerated workloads |
| NVDIMM (Large Memory) | 80 | 4 TB | 3 | For memory-intensive workloads |

The choice of node type affects both performance and cost. A structural analysis that fits in 192 GB runs well on the widely available SKX nodes. A coastal simulation with a dense mesh that benefits from fast memory access may run noticeably faster on SPR nodes. A machine-learning training job belongs on PVC nodes.

## Queues

Every TACC system organizes its nodes into **queues** (also called partitions). A queue groups nodes with similar hardware and enforces limits on how many nodes can be requested and how long a job can run. Different queues charge different rates.

Choosing the right queue affects both cost and wait time. Development queues have strict limits but start quickly, making them ideal for testing. Production queues allow more resources but may have longer wait times.

### Stampede3

The full queue policy is in the [Stampede3 User Guide](https://docs.tacc.utexas.edu/hpc/stampede3/#queues).

| Queue | Node Type | Max Nodes (cores) | Max Duration | Max Jobs in Queue | Charge Rate |
|---|---|---|---|---|---|
| nvdimm | ICX | 1 (80) | 48 hrs | 3 | 4 SUs |
| icx | ICX | 32 (2,560) | 48 hrs | 12 | 1.5 SUs |
| spr | SPR | 32 (3,584) | 48 hrs | 24 | 2 SUs |
| pvc | PVC (GPU) | 4 (384) | 48 hrs | 2 | 3 SUs |
| skx | SKX | 256 (12,288) | 48 hrs | 40 | 1 SU |
| skx-dev | SKX | 16 (768) | 2 hrs | 1 running, 2 total | 1 SU |

The `skx-dev` queue is designed for short test runs. Wait times are typically low. Live queue status is available on the [TACC system status page](https://tacc.utexas.edu/portal/system-status/Stampede3).

### Frontera

Each node has 56 Intel Cascade Lake cores and 192 GB RAM. Full policy in the [Frontera User Guide](https://docs.tacc.utexas.edu/hpc/frontera/#queues).

| Queue | Max Nodes | Max Duration | Notes |
|---|---|---|---|
| normal | 512 | 48 hrs | General production queue |
| development | 40 | 2 hrs | Testing and debugging |
| large | 2048 | 48 hrs | Requires special approval |
| flex | 128 | 48 hrs | Preemptable, lower priority |

### Lonestar6

Each node has 128 AMD Milan cores and 256 GB RAM. Full policy in the [Lonestar6 User Guide](https://docs.tacc.utexas.edu/hpc/lonestar6/#queues).

| Queue | Max Nodes | Max Duration | Notes |
|---|---|---|---|
| normal | 32 | 48 hrs | General production queue |
| development | 4 | 2 hrs | Testing and debugging |
| gpu-a100 | 16 | 48 hrs | GPU nodes with NVIDIA A100 |
| gpu-a100-dev | 4 | 2 hrs | GPU development queue |

## Allocations and Service Units

A TACC **allocation** is a grant of computing time tied to a research project. Running jobs charges **Service Units (SUs)** against that allocation. When the SUs run out, job submissions fail until more are requested.

The billing formula:

```
SUs = nodes x hours x charge_rate
```

A job running on 4 SKX nodes for 2 hours at 1 SU per node-hour costs 4 x 2 x 1 = 8 SUs. The same job on 4 SPR nodes at 2 SUs per node-hour costs 16 SUs.

Nodes are billed entirely, regardless of how many cores are used. A job using 4 cores on a 48-core SKX node still pays for the full node. Every job incurs a minimum charge of 15 minutes. Jobs that finish early are only charged for the time actually used.

Remaining SU balance and allocation codes are available on the [TACC Dashboard](https://tacc.utexas.edu/portal/dashboard).

## Resource sizing guidance

**Start small.** Run the model in the development queue with a short walltime and a single node. A 10-minute test on `skx-dev` costs almost nothing and catches most configuration errors before they waste hours of allocation time.

**Account for staging time.** Tapis copies input files before the job starts and archives outputs afterward. Both count against the walltime limit. If a simulation takes 30 minutes but staging takes 10, request at least 50 minutes.

**Match cores to the problem.** For [MPI](https://www.mpi-forum.org/) jobs, total ranks = node_count x cores_per_node. If a model is decomposed into 96 subdomains, request 2 nodes with 48 cores each. For serial jobs or [PyLauncher](https://github.com/TACC/pylauncher) sweeps, one node is usually enough.

**Watch memory per core.** All cores on a node share the same RAM. If each MPI process needs 8 GB and the node has 192 GB, running all 48 cores gives only about 4 GB per process. Requesting fewer cores per node gives each process more memory.

| Cores per Node (192 GB SKX) | Memory per Core |
|---|---|
| 48 | ~4 GB |
| 24 | 8 GB |
| 12 | 16 GB |
