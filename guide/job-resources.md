# Running HPC Jobs

## How a job moves through the system

<img src="../images/ComputeWorkflow.jpg" alt="DesignSafe HPC computational workflow" width="75%" />

An HPC job on [DesignSafe](https://designsafe-ci.org) passes through three stages.

1. Describe the job. From [JupyterHub](https://jupyter.designsafe-ci.org), the [web portal](https://www.designsafe-ci.org/rw/workspace/), or a Python script using [dapi](https://designsafe-ci.github.io/dapi/), specify the application, input files, and resource requirements (nodes, cores, walltime).

2. [Tapis](https://tapis.readthedocs.io/en/latest/) handles the logistics. Tapis copies input files to the execution system, generates a [SLURM](https://slurm.schedmd.com/documentation.html) batch script matching the requested resources, submits it to the scheduler, and monitors execution.

3. [TACC](https://www.tacc.utexas.edu/) runs the computation. SLURM places the job in a queue. When the requested nodes become available, it launches the application. After completion, Tapis copies the results back to DesignSafe storage.

In detail, the sequence is

1. Call `ds.jobs.submit()` in dapi, or click Submit in the web portal.
2. Tapis copies input files from DesignSafe storage to the execution system.
3. Tapis generates a SLURM batch script and submits it.
4. SLURM queues the job. When nodes become available, it launches the application.
5. The application runs and writes output to a working directory on the compute system.
6. Tapis archives the output back to DesignSafe storage.
7. Retrieve results with `job.list_outputs()` and `job.get_output_content()`.

The wait between steps 3 and 4 is queue time. It depends on how many resources were requested, how busy the system is, and which queue was selected. Development queues typically start within minutes. A 64-node production job on the normal queue may wait longer.

## Choosing resources

Submitting a job means requesting a share of a supercomputer's hardware for a limited time. Getting the request right matters. Too little time or memory causes the job to fail. Too many nodes wastes allocation budget and increases queue wait time. The parameters below control this balance.

## Parameters

| Parameter | dapi argument | Effect |
|---|---|---|
| Allocation | `allocation` | The TACC project account charged for the job |
| Queue | `queue` | Partition the job runs in, affecting max nodes, time limits, and priority |
| Max runtime | `max_minutes` | Wall-clock time limit in minutes (SLURM kills jobs that exceed it) |
| Node count | `node_count` | Number of physical machines allocated |
| Cores per node | `cores_per_node` | CPU cores requested per node (total cores = node_count x cores_per_node) |
| Memory | `memory_mb` | Memory in megabytes (optional, since system defaults usually suffice) |

## How dapi Maps to SLURM

Calling `ds.jobs.generate()` builds a Tapis job request. Tapis then generates a SLURM batch script with `#SBATCH` directives that match the parameters.

```python
job_request = ds.jobs.generate(
    app_id="opensees-mp-s3",
    input_dir_uri=input_uri,
    script_filename="model.tcl",
    node_count=2,
    cores_per_node=48,
    max_minutes=120,
    allocation="your_allocation",
    queue="skx",
)
```

This produces a SLURM script equivalent to

```bash
#!/bin/bash
#SBATCH -A your_allocation
#SBATCH -p skx
#SBATCH -N 2
#SBATCH -n 96
#SBATCH -t 02:00:00
# ... module loads, execution commands ...
```

The script is generated automatically. There is no need to write it by hand.

## Stampede3 node types

Stampede3 is the primary execution system for DesignSafe HPC jobs. It contains several node types with different capabilities.

| Node Type | Cores per Node | Memory | Count | Notes |
|---|---|---|---|---|
| ICX (Ice Lake) | 80 | 256 GB DDR4 | 224 | Standard compute nodes |
| SPR (Sapphire Rapids) | 112 | 128 GB HBM2e | 560 | High memory bandwidth per core, good for memory-bound applications |
| SKX (Skylake) | 48 | 192 GB DDR4 | 1,060 | Older generation, most numerous |
| PVC (Ponte Vecchio) | 96 | 512 GB DDR5 | 20 | Intel GPUs with 128 GB HBM2e each, for ML and GPU-accelerated workloads |
| NVDIMM (Large Memory) | 80 | 4 TB | 3 | For memory-intensive workloads |

The choice of node type affects both performance and cost. A structural analysis that fits in 192 GB runs well on the widely available SKX nodes. A coastal simulation with a dense mesh that benefits from fast memory access may run noticeably faster on SPR nodes with HBM2e. A machine-learning training job belongs on PVC nodes with their GPU accelerators.

## Queues

Every TACC system organizes its compute nodes into queues (also called partitions). A queue groups a set of nodes with similar hardware and enforces limits on how many nodes can be requested, how long a job can run, and how many jobs a single user can have active at once. Different queues also charge different SU rates per node-hour.

Choosing the right queue affects both cost and wait time. Smaller queues with shorter time limits tend to have lower wait times because the scheduler can fit them into gaps between larger jobs. Every system also has a development queue with strict limits, designed for quick test runs with minimal waiting.

When the `queue` parameter is set in dapi, Tapis passes it to SLURM as the `-p` (partition) flag.

### [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/)

Stampede3 has several node types, each with its own queue. The full queue policy is documented in the [Stampede3 User Guide](https://docs.tacc.utexas.edu/hpc/stampede3/#queues).

| Queue | Node Type | Max Nodes (cores) | Max Duration | Max Jobs in Queue | Charge Rate |
|---|---|---|---|---|---|
| nvdimm | ICX | 1 (80) | 48 hrs | 3 | 4 SUs |
| icx | ICX | 32 (2,560) | 48 hrs | 12 | 1.5 SUs |
| spr | SPR | 32 (3,584) | 48 hrs | 24 | 2 SUs |
| pvc | PVC (GPU) | 4 (384) | 48 hrs | 2 | 3 SUs |
| skx | SKX | 256 (12,288) | 48 hrs | 40 | 1 SU |
| skx-dev | SKX | 16 (768) | 2 hrs | 1 running, 2 total | 1 SU |

The `skx-dev` queue is intended for short testing and debugging runs. Wait times are typically low, but only one job can run at a time with at most two total in the queue. Live queue status is available on the [TACC system status page](https://tacc.utexas.edu/portal/system-status/Stampede3).

### [Frontera](https://docs.tacc.utexas.edu/hpc/frontera/)

Frontera is TACC's flagship system for large-scale computation. Each compute node has 56 Intel Cascade Lake cores and 192 GB of RAM. The full queue policy is in the [Frontera User Guide](https://docs.tacc.utexas.edu/hpc/frontera/#queues).

| Queue | Max Nodes | Max Duration | Notes |
|---|---|---|---|
| normal | 512 | 48 hrs | General production queue |
| development | 40 | 2 hrs | Testing and debugging |
| large | 2048 | 48 hrs | Requires special approval |
| flex | 128 | 48 hrs | Preemptable, lower priority |

### [Lonestar6](https://docs.tacc.utexas.edu/hpc/lonestar6/)

Lonestar6 has 128-core AMD Milan nodes with 256 GB of RAM. The full queue policy is in the [Lonestar6 User Guide](https://docs.tacc.utexas.edu/hpc/lonestar6/#queues).

| Queue | Max Nodes | Max Duration | Notes |
|---|---|---|---|
| normal | 32 | 48 hrs | General production queue |
| development | 4 | 2 hrs | Testing and debugging |
| gpu-a100 | 16 | 48 hrs | GPU nodes with NVIDIA A100 |
| gpu-a100-dev | 4 | 2 hrs | GPU development queue |

## Allocations and Service Units

A TACC allocation is a grant of computing resources tied to a project. Running jobs charges Service Units (SUs) against that allocation. When the SUs are exhausted, job submissions will fail until more are requested.

The billing formula is

```
SUs = nodes x hours x charge_rate
```

A job running on 4 SKX nodes for 2 hours at a charge rate of 1 SU per node-hour costs 4 x 2 x 1 = 8 SUs. The same job on 4 SPR nodes at 2 SUs per node-hour would cost 16 SUs.

Several billing rules apply on Stampede3. SLURM measures usage to the nearest few seconds, so a job that ends early and exits cleanly is only charged for the time actually used. Nodes are not shared, meaning the entire node is billed regardless of how many cores are used. Every job incurs a minimum charge of 15 minutes. Different queues charge different SU rates per node-hour.

Remaining SU balance and allocation codes are available on the [TACC Dashboard](https://tacc.utexas.edu/portal/dashboard). Usage figures may lag slightly behind actual job activity.

## Resource Sizing Guidance

Start small in the development queue. Run the model with a short walltime and a single node to confirm it works before scaling up. A 10-minute test run on `skx-dev` costs almost nothing and catches most configuration errors before they waste hours of allocation time.

Walltime includes file staging. Tapis copies input files to the compute system before the job starts and archives outputs afterward. Both transfers count against the walltime limit. If a simulation takes 30 minutes but staging takes 10, request at least 50 minutes.

Match cores to the domain decomposition for [MPI](https://www.mpi-forum.org/) jobs. Total MPI ranks = node_count x cores_per_node. If a model is decomposed into 96 subdomains, request 2 nodes with 48 cores each.

Memory-intensive jobs may need fewer cores per node. All cores on a node share the same RAM pool. If each MPI process needs 8 GB and the node has 192 GB, running all 48 cores gives only about 4 GB per process. Requesting fewer cores per node gives each process more memory headroom.

| Processes per Node (192 GB SKX) | Memory per Process |
|---|---|
| 48 | ~4 GB |
| 24 | 8 GB |
| 12 | 16 GB |

For serial jobs or [PyLauncher](https://github.com/TACC/pylauncher) sweeps, one node is usually sufficient. Request enough cores to match the number of concurrent tasks PyLauncher should run. A sweep of 48 independent OpenSeesPy analyses fits neatly on a single SKX node with 48 cores.
