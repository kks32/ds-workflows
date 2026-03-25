# Compute Environments

Where does a job actually run? The answer depends on its size and how much interaction it needs. A quick test of a Tcl model can finish on a shared virtual machine in seconds. A 3D nonlinear time-history analysis of a 40-story building may need 128 cores on [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/) for several hours. A post-processing script that plots spectral acceleration curves fits comfortably inside a Jupyter notebook. [DesignSafe](https://designsafe-ci.org) provides three compute environments, and many researchers move between them as a project progresses from early development to production runs.

## JupyterHub on Kubernetes

The DesignSafe [JupyterHub](https://jupyter.designsafe-ci.org) runs on a [Kubernetes](https://kubernetes.io/) cluster at [TACC](https://www.tacc.utexas.edu/). When a researcher starts a session, Kubernetes provisions a dedicated container with up to 8 CPU cores and 20 GB of RAM. These resources belong exclusively to that session. No other users share the container's CPUs or memory, though the underlying physical node hosts multiple containers, so I/O may see minor contention under heavy load.

Sessions start immediately with no queue wait.

JupyterHub is the right environment for developing and testing workflows, running Python scripts interactively, pre-processing input files, visualizing simulation output, and submitting jobs to HPC through [Tapis](https://tapis.readthedocs.io/en/latest/) and [dapi](https://designsafe-ci.github.io/dapi/). When a workload needs more memory, more cores, or multi-node execution, the job should move to HPC.

## Virtual Machines

DesignSafe provides access to shared virtual machines (VMs) at TACC for several applications. A VM is a simulated computer running on physical hardware, sharing that hardware's resources with other VMs. VM jobs bypass the HPC queue and typically start immediately, but performance varies under load because the hardware is shared across users.

| Application | VM Type | Notes |
|---|---|---|
| [OpenSees](https://opensees.berkeley.edu/) EXPRESS | Submit-only (no SSH) | Sequential Tcl jobs, 24 cores, 48 GB RAM |
| OpenSees Interactive | Interactive IDE | All OpenSees variants (Tcl and Python), 24 cores, 48 GB RAM |
| [MATLAB](https://www.mathworks.com/products/matlab.html) | Interactive session | MATLAB 2022b, suited to lighter workloads |
| [ADCIRC](https://adcirc.org/) Interactive | Interactive JupyterLab | Pre-compiled ADCIRC, testing and development |
| [STKO](https://asdeasoft.net/stko/) | Interactive desktop (VNC) | OpenSees visualization and input/output file creation |
| [QGIS](https://qgis.org/) | Interactive desktop (VNC) | Geographic information system for spatial data |

Some VM applications (STKO, QGIS) provide a full graphical desktop through [VNC](https://en.wikipedia.org/wiki/Virtual_Network_Computing) (Virtual Network Computing). VNC streams a remote desktop to the browser, so researchers can interact with GUI-based tools as if they were running locally. The session opens directly from the DesignSafe portal after a short startup period. [DCV](https://docs.tacc.utexas.edu/tutorials/remotedesktopaccess/) (Desktop Cloud Visualization) is a similar remote desktop technology available for certain applications on Frontera.

VMs work well for lightweight, short-running tasks and interactive exploration. A researcher testing a 5-second ground-motion analysis in OpenSees, running a quick MATLAB curve-fitting script, building a finite-element mesh in STKO, or inspecting a geospatial dataset in QGIS can get results without waiting in a queue. For large or parallel computations, HPC is a better fit.

## HPC at TACC

For production-scale simulations, DesignSafe connects to TACC's high-performance computing systems. These are clusters of interconnected machines (nodes) that support multi-node execution, [MPI](https://www.mpi-forum.org/) parallelism, and large memory allocations. Jobs are submitted through [SLURM](https://slurm.schedmd.com/documentation.html), the scheduler that manages all compute resources, and wait in a queue until the requested hardware becomes available.

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

[Frontera](https://docs.tacc.utexas.edu/hpc/frontera/) is TACC's leadership-class system with 56-core Intel Cascade Lake nodes and 192 GB of RAM, designed for the largest parallel workloads. [Lonestar6](https://docs.tacc.utexas.edu/hpc/lonestar6/) provides general-purpose HPC with 128-core AMD Milan nodes and 256 GB of RAM, including GPU nodes with NVIDIA A100 accelerators. Both systems are accessible through DesignSafe via Tapis. Consult the linked TACC user guides for queue policies and allocation details.

## Choosing an Environment

| Situation | Recommended Environment | Example |
|---|---|---|
| Developing scripts, testing small models, visualizing results | JupyterHub | Writing a Python post-processing script, plotting response spectra |
| Running a quick interactive session | VM | Testing an OpenSees Tcl model, running a short MATLAB analysis, exploring spatial data in QGIS, building a mesh in STKO |
| Running a large or long simulation | HPC (Stampede3, Frontera, LS6) | A nonlinear time-history analysis of a 3D building, an ADCIRC storm-surge forecast |
| Running hundreds of independent simulations | HPC with [PyLauncher](https://github.com/TACC/pylauncher) | A fragility study varying ground-motion intensity across 500 records |
| Parallel simulation with domain decomposition | HPC with MPI | A multi-node OpenFOAM CFD simulation, an ADCIRC mesh with millions of elements |

Many researchers follow a natural progression. Develop and test interactively in JupyterHub, validate with small problems on a VM or in the development queue, then scale to HPC batch jobs for production runs.

[Running HPC Jobs](job-resources.md) covers job submission, resource selection, and queue policies in detail.
