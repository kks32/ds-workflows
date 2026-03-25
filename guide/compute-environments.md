# Compute Environments

This page covers the hardware details behind each environment introduced in [How It Works](how-it-works.md).

## JupyterHub on Kubernetes

The DesignSafe [JupyterHub](https://jupyter.designsafe-ci.org) runs on a [Kubernetes](https://kubernetes.io/) cluster at [TACC](https://www.tacc.utexas.edu/). When a researcher starts a session, Kubernetes provisions a dedicated container with up to 8 CPU cores and 20 GB of RAM. These resources belong exclusively to that session. No other users share the container's CPUs or memory, though the underlying physical node hosts multiple containers, so I/O may see minor contention under heavy load.

Sessions start immediately with no queue wait.

For heavier interactive work, Jupyter HPC Native sessions run directly on [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/) CPU nodes or Vista H200 GPU nodes. These provide access to full node resources and support persistent conda environments, but they go through the [SLURM](https://slurm.schedmd.com/documentation.html) queue and can run up to 48 hours.

## Virtual Machines

[DesignSafe](https://designsafe-ci.org) provides access to shared virtual machines (VMs) at TACC for several applications. A VM is a simulated computer running on physical hardware, sharing that hardware's resources with other VMs. VM jobs bypass the HPC queue and typically start immediately, but performance varies under load because the hardware is shared across users.

| Application | VM Type | Notes |
|---|---|---|
| [OpenSees](https://opensees.berkeley.edu/) EXPRESS | Submit-only (no SSH) | Sequential Tcl jobs, 24 cores, 48 GB RAM |
| OpenSees Interactive | Interactive IDE | All OpenSees variants (Tcl and Python), 24 cores, 48 GB RAM |
| [MATLAB](https://www.mathworks.com/products/matlab.html) | Interactive session | MATLAB 2022b, suited to lighter workloads |
| [ADCIRC](https://adcirc.org/) Interactive | Interactive JupyterLab | Pre-compiled ADCIRC, testing and development |
| [STKO](https://asdeasoft.net/stko/) | Interactive desktop (DCV) | OpenSees visualization and input/output file creation |
| [QGIS](https://qgis.org/) | Interactive desktop (DCV) | Geographic information system for spatial data |

STKO and QGIS provide a full graphical desktop through [NICE DCV](https://docs.tacc.utexas.edu/tutorials/remotedesktopaccess/) (Desktop Cloud Visualization). DCV streams a remote desktop to the browser, so researchers can interact with GUI-based tools as if they were running locally. The session opens directly from the DesignSafe portal after a short startup period.

## HPC at TACC

For production-scale simulations, DesignSafe connects to TACC's high-performance computing systems. These are clusters of interconnected machines (nodes) that support multi-node execution, [MPI](https://www.mpi-forum.org/) parallelism, and large memory allocations. Jobs are submitted through SLURM, the scheduler that manages all compute resources, and wait in a queue until the requested hardware becomes available.

### Stampede3

Stampede3 is the primary execution system for DesignSafe HPC jobs. It contains several node types with different capabilities.

| Node Type | Cores per Node | Memory | Count | Notes |
|---|---|---|---|---|
| ICX (Ice Lake) | 80 | 256 GB DDR4 | 224 | Standard compute nodes |
| SPR (Sapphire Rapids) | 112 | 128 GB HBM2e | 560 | High memory bandwidth per core, good for memory-bound applications |
| SKX (Skylake) | 48 | 192 GB DDR4 | 1,060 | Older generation, most numerous |
| PVC (Ponte Vecchio) | 96 | 512 GB DDR5 | 20 | Intel GPUs with 128 GB HBM2e each, for ML and GPU-accelerated workloads |
| NVDIMM (Large Memory) | 80 | 4 TB | 3 | For memory-intensive workloads |

The choice of node type affects both performance and cost. A structural analysis that fits in 192 GB runs well on the widely available SKX nodes. A coastal simulation with a dense mesh that benefits from fast memory access may run noticeably faster on SPR nodes with HBM2e. A machine-learning training job belongs on PVC nodes with their GPU accelerators.

### Frontera and Lonestar6

[Frontera](https://docs.tacc.utexas.edu/hpc/frontera/) is TACC's leadership-class system with 56-core Intel Cascade Lake nodes and 192 GB of RAM, designed for the largest parallel workloads. [Lonestar6](https://docs.tacc.utexas.edu/hpc/lonestar6/) provides general-purpose HPC with 128-core AMD Milan nodes and 256 GB of RAM, including GPU nodes with NVIDIA A100 accelerators. Both systems are accessible through DesignSafe via [Tapis](https://tapis.readthedocs.io/en/latest/). Consult the linked TACC user guides for queue policies and allocation details.

[Running HPC Jobs](job-resources.md) covers job submission, resource selection, and queue policies in detail.
