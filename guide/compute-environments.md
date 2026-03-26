# Compute Environments

DesignSafe provides three places where computation happens. Each serves a different purpose, and most researchers move between them as a project evolves: develop interactively in JupyterHub, then submit production runs to HPC.

## JupyterHub

[JupyterHub](https://jupyter.designsafe-ci.org) is where most day-to-day work happens. Each session gets a dedicated Kubernetes container at [TACC](https://www.tacc.utexas.edu/) with up to **8 CPU cores** and **20 GB RAM**. Sessions start immediately with no queue wait. The browser-based environment includes notebooks, a terminal, a file manager, and a text editor — all sharing the same filesystem.

Move to HPC when the workload needs more memory, more cores, multi-node parallelism (MPI), or longer runtimes than an interactive session allows.

For heavier interactive work, **Jupyter HPC Native** sessions run directly on [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/) or Vista GPU nodes with full node resources. These go through the SLURM queue, so there may be a wait before the session starts.

## Virtual machines

Virtual machines (VMs) run interactive GUI applications without a queue wait. [OpenSees](https://opensees.berkeley.edu/) Interactive, [MATLAB](https://www.mathworks.com/products/matlab.html), [ADCIRC](https://adcirc.org/) Interactive, [STKO](https://asdeasoft.net/stko/), and [QGIS](https://qgis.org/) all run on shared VMs at TACC. STKO and QGIS provide a full graphical desktop through [NICE DCV](https://docs.tacc.utexas.edu/tutorials/remotedesktopaccess/), which streams a remote desktop to the browser. VMs share hardware across users, so they work best for lightweight tasks and quick tests.

## HPC systems

HPC (High-Performance Computing) systems handle production-scale computation. These are clusters of interconnected machines (**nodes**), each with dozens of CPU cores and hundreds of gigabytes of memory. [SLURM](https://slurm.schedmd.com/documentation.html) manages the job queue. Long-running simulations, multi-core parallel analyses, and parametric sweeps with hundreds of runs all belong on HPC.

DesignSafe researchers have access to three TACC systems:

| System | Cores per Node | Memory per Node | Primary Use |
|---|---|---|---|
| [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/) | 48–112 (varies by node type) | 128 GB–4 TB | General-purpose, most DesignSafe jobs |
| [Frontera](https://docs.tacc.utexas.edu/hpc/frontera/) | 56 | 192 GB | Large-scale parallel simulations |
| [Lonestar6](https://docs.tacc.utexas.edu/hpc/lonestar6/) | 128 | 256 GB | General-purpose, GPU nodes available |

### Stampede3 node types

| Node Type | Cores per Node | Memory | Count | Notes |
|---|---|---|---|---|
| SKX (Skylake) | 48 | 192 GB DDR4 | 1,060 | Most numerous, good default choice |
| ICX (Ice Lake) | 80 | 256 GB DDR4 | 224 | Standard compute nodes |
| SPR (Sapphire Rapids) | 112 | 128 GB HBM2e | 560 | High memory bandwidth |
| PVC (Ponte Vecchio) | 96 | 512 GB DDR5 | 20 | Intel GPUs for ML workloads |
| NVDIMM (Large Memory) | 80 | 4 TB | 3 | Memory-intensive workloads |

### Nodes, cores, and memory

A **node** is a complete physical computer. Each node has multiple **cores** (CPUs) that execute work in parallel, sharing the same pool of RAM.

When submitting a job, specify `node_count`, `cores_per_node`, and `max_minutes`. Total cores = node_count x cores_per_node. For [MPI](https://www.mpi-forum.org/) jobs, each core runs one parallel process (rank). For [PyLauncher](https://github.com/TACC/pylauncher) sweeps, each core runs one independent task.

All cores on a node share memory. If each process needs more memory, request fewer cores per node:

| Cores per Node (192 GB SKX) | Memory per Core |
|---|---|
| 48 | ~4 GB |
| 24 | ~8 GB |
| 12 | ~16 GB |

## SLURM and queues

SLURM is the job scheduler on all TACC systems. When a job is submitted, SLURM places it in a **queue** (also called a partition). Each queue groups nodes with similar hardware and enforces limits on node count and runtime. Researchers using DesignSafe never write SLURM scripts directly — [Tapis](https://tapis.readthedocs.io/en/latest/) generates them automatically from the job parameters.

**Stampede3 queues** (full policy in the [Stampede3 User Guide](https://docs.tacc.utexas.edu/hpc/stampede3/#queues)):

| Queue | Node Type | Max Nodes | Max Duration | Charge Rate |
|---|---|---|---|---|
| skx | SKX (48 cores, 192 GB) | 256 | 48 hrs | 1 SU |
| skx-dev | SKX | 16 | 2 hrs | 1 SU |
| icx | ICX (80 cores, 256 GB) | 32 | 48 hrs | 1.5 SUs |
| spr | SPR (112 cores, 128 GB HBM) | 32 | 48 hrs | 2 SUs |
| pvc | PVC (GPU) | 4 | 48 hrs | 3 SUs |
| nvdimm | ICX (80 cores, 4 TB) | 1 | 48 hrs | 4 SUs |

The `skx-dev` queue is designed for short test runs with low wait times. Always test there before submitting production jobs.

**Frontera queues** (56 cores, 192 GB per node; [Frontera User Guide](https://docs.tacc.utexas.edu/hpc/frontera/#queues)):

| Queue | Max Nodes | Max Duration | Notes |
|---|---|---|---|
| normal | 512 | 48 hrs | General production |
| development | 40 | 2 hrs | Testing and debugging |
| large | 2048 | 48 hrs | Requires special approval |

**Lonestar6 queues** (128 cores, 256 GB per node; [Lonestar6 User Guide](https://docs.tacc.utexas.edu/hpc/lonestar6/#queues)):

| Queue | Max Nodes | Max Duration | Notes |
|---|---|---|---|
| normal | 32 | 48 hrs | General production |
| development | 4 | 2 hrs | Testing and debugging |
| gpu-a100 | 16 | 48 hrs | NVIDIA A100 GPU nodes |
| gpu-a100-dev | 4 | 2 hrs | GPU development |

### Choosing a queue

1. **Estimate memory per process.** If each MPI rank needs 8 GB and the node has 192 GB, use at most 24 cores per node.
2. **Determine total cores needed.** A model decomposed into 96 subdomains needs 96 cores (e.g., 2 nodes x 48 cores).
3. **Pick the queue that fits.** Use `skx-dev` or `development` for testing. Use production queues for real runs.
4. **Check system load.** Live queue status: [Stampede3](https://tacc.utexas.edu/portal/system-status/Stampede3), [Frontera](https://tacc.utexas.edu/portal/system-status/Frontera), [Lonestar6](https://tacc.utexas.edu/portal/system-status/Lonestar6).

## Allocations and Service Units

A TACC **allocation** is a grant of computing time tied to a research project. Running jobs charges **Service Units (SUs)**:

```
SUs = nodes x hours x charge_rate
```

A job on 4 SKX nodes for 2 hours at 1 SU/node-hour costs 8 SUs. The same job on SPR nodes at 2 SUs/node-hour costs 16 SUs. Nodes are billed entirely regardless of how many cores are used. Every job incurs a minimum charge of 15 minutes.

HPC-enabled tools (OpenSeesMP, OpenFOAM, ADCIRC) require an allocation. If you don't have one, submit a ticket through the [DesignSafe help desk](https://www.designsafe-ci.org/help/new-ticket/). Remaining balance and allocation codes are on the [TACC Dashboard](https://tacc.utexas.edu/portal/dashboard).
