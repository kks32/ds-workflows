# Computational Workflows on DesignSafe

[DesignSafe](https://designsafe-ci.org) provides a comprehensive cyberinfrastructure for conducting, managing, and analyzing research workflows in natural hazards engineering. It brings together interactive computing environments, shared data services, and large-scale computational resources to support the full research lifecycle, from early model development to large ensemble simulations to advanced post-processing and data analysis.

At its core, DesignSafe is tightly integrated with the [Texas Advanced Computing Center (TACC)](https://www.tacc.utexas.edu/), which supplies the high-performance computing (HPC) systems and large-scale storage needed to execute demanding analyses. Most computational jobs launched through DesignSafe, whether from the web portal, a Jupyter notebook, or an automated pipeline, ultimately run on TACC systems such as [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/), enabling simulations and studies that would be impractical on a laptop.

The platform supports lightweight interactive exploration as well as production-scale simulations spanning many nodes and thousands of CPU cores. Data and compute are co-located at TACC, so earthquake ground-motion records, structural response datasets, and geotechnical field data hosted on DesignSafe can be accessed directly from the compute environment without downloading and re-uploading. A simulation that takes 12 hours on a laptop can finish in minutes by distributing the workload across many cores. Understanding how to move effectively between interactive and batch modes is essential for building efficient and reproducible research workflows.

## The DesignSafe Portal

Everything starts at the [DesignSafe web portal](https://www.designsafe-ci.org/rw/workspace/). The portal is the central hub for launching tools, managing data, and submitting jobs. From the portal workspace, researchers can

- Launch [JupyterHub](https://jupyter.designsafe-ci.org) for interactive notebooks (Python, R, Julia)
- Launch Jupyter HPC Native sessions on Stampede3 or Vista GPU nodes for heavier interactive work
- Open interactive desktop sessions via DCV for tools like [STKO](https://asdeasoft.net/stko/) and [QGIS](https://qgis.org/), or launch visualization tools like [ParaView](https://www.paraview.org/) and [VisIt](https://visit-dav.github.io/visit-website/) on HPC nodes
- Submit HPC batch jobs for simulation applications ([OpenSees](https://opensees.berkeley.edu/), [OpenFOAM](https://www.openfoam.com/), [ADCIRC](https://adcirc.org/), [LS-DYNA](https://lsdyna.ansys.com/), MPM, and others)
- Access the [Data Depot](https://designsafe-ci.org/user-guide/datadepot/) to browse, upload, share, curate, and publish datasets

The portal's submission forms look interactive, but jobs submitted to HPC systems are batch jobs. The form simplifies the process of specifying inputs, resources, and parameters, but execution is queued, scheduled, and non-interactive.

Researchers can also bypass the portal entirely and submit jobs programmatically from a Jupyter notebook using [dapi](https://designsafe-ci.github.io/dapi/) or the [Tapis](https://tapis.readthedocs.io/en/latest/) Python client (`tapipy`). This is the preferred approach for automated pipelines, parameter sweeps, and reproducible workflows.

## Compute environments

DesignSafe provides three types of compute environments, all accessible from the portal.

<img src="../images/compute-environments.svg" alt="DesignSafe compute environments overview" width="100%" />

### JupyterHub

The DesignSafe JupyterHub runs on a [Kubernetes](https://kubernetes.io/) cluster at TACC. When a researcher starts a session, Kubernetes provisions a dedicated container with up to 8 CPU cores and 20 GB of RAM. These resources belong exclusively to that session. No other users share the container's CPUs or memory, though the underlying physical node hosts multiple containers, so I/O may see minor contention under heavy load.

Sessions start immediately with no queue wait.

JupyterHub is the right environment for developing and testing workflows, running Python scripts interactively, pre-processing input files, visualizing simulation output, and submitting jobs to HPC through Tapis and dapi.

For heavier interactive work, Jupyter HPC Native sessions run directly on Stampede3 CPU nodes or Vista H200 GPU nodes. These provide access to full node resources and support persistent conda environments, but they go through the [SLURM](https://slurm.schedmd.com/documentation.html) queue and can run up to 48 hours.

### Virtual machines

DesignSafe provides access to shared virtual machines (VMs) at TACC for several applications. A VM is a simulated computer running on physical hardware, sharing that hardware's resources with other VMs. VM jobs bypass the HPC queue and typically start immediately, but performance varies under load because the hardware is shared across users.

| Application | VM Type | Notes |
|---|---|---|
| OpenSees EXPRESS | Submit-only (no SSH) | Sequential Tcl jobs, 24 cores, 48 GB RAM |
| OpenSees Interactive | Interactive IDE | All OpenSees variants (Tcl and Python), 24 cores, 48 GB RAM |
| [MATLAB](https://www.mathworks.com/products/matlab.html) | Interactive session | MATLAB 2022b, suited to lighter workloads |
| ADCIRC Interactive | Interactive JupyterLab | Pre-compiled ADCIRC, testing and development |
| STKO | Interactive desktop (DCV) | OpenSees visualization and input/output file creation |
| QGIS | Interactive desktop (DCV) | Geographic information system for spatial data |

STKO and QGIS provide a full graphical desktop through [NICE DCV](https://docs.tacc.utexas.edu/tutorials/remotedesktopaccess/) (Desktop Cloud Visualization). DCV streams a remote desktop to the browser, so researchers can interact with GUI-based tools as if they were running locally. The session opens directly from the DesignSafe portal after a short startup period.

VMs work well for lightweight, short-running tasks and interactive exploration. A researcher testing a 5-second ground-motion analysis in OpenSees, running a quick MATLAB curve-fitting script, building a finite-element mesh in STKO, or inspecting a geospatial dataset in QGIS can get results without waiting in a queue. For large or parallel computations, HPC is a better fit.

### HPC at TACC

For production-scale simulations, DesignSafe connects to TACC's high-performance computing systems. These are clusters of interconnected machines (nodes) that support multi-node execution, [MPI](https://www.mpi-forum.org/) parallelism, and large memory allocations. Jobs are submitted through SLURM, the scheduler that manages all compute resources, and wait in a queue until the requested hardware becomes available. Long-running simulations, parallel analyses across many cores, and parametric sweeps with hundreds of runs all belong on HPC.

**Stampede3** is the primary execution system for DesignSafe HPC jobs. It contains several node types with different capabilities.

| Node Type | Cores per Node | Memory | Count | Notes |
|---|---|---|---|---|
| ICX (Ice Lake) | 80 | 256 GB DDR4 | 224 | Standard compute nodes |
| SPR (Sapphire Rapids) | 112 | 128 GB HBM2e | 560 | High memory bandwidth per core, good for memory-bound applications |
| SKX (Skylake) | 48 | 192 GB DDR4 | 1,060 | Older generation, most numerous |
| PVC (Ponte Vecchio) | 96 | 512 GB DDR5 | 20 | Intel GPUs with 128 GB HBM2e each, for ML and GPU-accelerated workloads |
| NVDIMM (Large Memory) | 80 | 4 TB | 3 | For memory-intensive workloads |

The choice of node type affects both performance and cost. A structural analysis that fits in 192 GB runs well on the widely available SKX nodes. A coastal simulation with a dense mesh that benefits from fast memory access may run noticeably faster on SPR nodes with HBM2e. A machine-learning training job belongs on PVC nodes with their GPU accelerators.

[Frontera](https://docs.tacc.utexas.edu/hpc/frontera/) is TACC's leadership-class system with 56-core Intel Cascade Lake nodes and 192 GB of RAM, designed for the largest parallel workloads. [Lonestar6](https://docs.tacc.utexas.edu/hpc/lonestar6/) provides general-purpose HPC with 128-core AMD Milan nodes and 256 GB of RAM, including GPU nodes with NVIDIA A100 accelerators. Both systems are accessible through DesignSafe via Tapis. Consult the linked TACC user guides for queue policies and allocation details.

## Interactive and batch workflows

Computational research on DesignSafe falls into two modes of work.

Interactive workflows involve writing code, running it, inspecting results, modifying inputs, and iterating. The feedback loop is tight, often seconds between runs. JupyterHub, the VM-based sessions (MATLAB, OpenSees Interactive, QGIS), and DCV desktop environments (STKO, QGIS) all support this mode.

Batch workflows involve submitting a job to a queue and collecting results later. There is no live interaction during execution. The job runs on dedicated hardware, often for minutes to hours, and Tapis archives the output when it finishes. All HPC jobs on Stampede3, Frontera, and Lonestar6 are batch jobs, whether submitted through the portal, dapi, or direct SSH.

Most research projects use both modes. A typical progression is to develop and test interactively in JupyterHub, then submit production runs as batch jobs to HPC.

## Designing a workflow

A computational workflow should be designed around the research question, not around a specific tool or computing system. A workflow that works for exploratory analysis may fail when scaled to thousands of simulations or extended to include coupled physics and uncertainty quantification.

An effective workflow is modular, with each stage handling a well-defined task.

1. **Input generation** prepares models, parameters, ground motions, or meshes.
2. **Execution** runs the simulation, ensemble, or training loop.
3. **Post-processing** extracts results, computes statistics, and generates figures.
4. **Iteration** repeats execution across parameter sets, Monte Carlo samples, or convergence loops.

When these stages are cleanly separated, each one can be reused across projects, swapped to a different execution environment as computational demands grow, or combined in new ways as the research evolves. A mesh generator can change without touching the solver. A plotting script can be updated without re-running simulations. The execution stage can move from JupyterHub to HPC batch without rewriting the input generation or post-processing steps.

Scalability also needs to be considered from the start. A workflow that runs successfully for one model, one parameter set, and one dataset should be able to scale to hundreds of simulations, higher-resolution models, or more complex coupling. Different workloads scale in different ways. Some (parameter sweeps, Monte Carlo) scale trivially by running many independent jobs. Others (large finite-element models, coupled simulations) are limited by memory per node or inter-process communication. Some benefit from GPUs. Matching the workload to the right execution strategy is as important as choosing the right hardware.

## Choosing the right environment

| Situation | Environment | Example |
|---|---|---|
| Writing and testing code, visualizing results | JupyterHub (interactive) | Developing a Python post-processing script, plotting response spectra |
| Quick interactive session with a GUI tool | VM with DCV (interactive) | Exploring spatial data in QGIS, building an OpenSees model in STKO |
| Quick serial simulation test | VM (interactive or submit-only) | Testing an OpenSees Tcl model, running a short MATLAB analysis |
| Large or long-running simulation | HPC batch job | Nonlinear time-history analysis of a 3D building, ADCIRC storm-surge forecast |
| Hundreds of independent simulations | HPC batch with [PyLauncher](https://github.com/TACC/pylauncher) | Fragility study varying ground-motion intensity across 500 records |
| Parallel simulation needing many cores | HPC batch with MPI | Multi-node OpenFOAM CFD, ADCIRC mesh with millions of elements |
| GPU-accelerated computation or AI/ML | Jupyter HPC Native (Vista) or HPC GPU queue | Training a neural network, GPU-accelerated material simulation |

## Storage and data management

One of DesignSafe's key advantages is that data and compute live in the same place. Research datasets, simulation inputs, and published results are all stored at TACC alongside the compute systems. A ground-motion database in CommunityData, for example, can be referenced directly from a simulation job without downloading it to a laptop and re-uploading it to the cluster.

The [Data Depot](https://designsafe-ci.org/user-guide/datadepot/) provides the web interface for managing files across these storage areas.

| Storage area | What it holds | Backed up | Accessible from |
|---|---|---|---|
| MyData | Private files | Yes | JupyterHub, portal, VMs |
| MyProjects | Shared project files | Yes | JupyterHub, portal, collaborators |
| Work | Active HPC data | No | Compute nodes, JupyterHub |
| Scratch | Temporary fast storage | No (purged periodically) | Compute nodes |
| CommunityData | Public datasets | Yes | Everyone (read-only) |
| Published | Archived datasets with DOIs | Yes | Everyone (read-only) |

MyData, MyProjects, CommunityData, and Published all live on Corral, TACC's backed-up networked storage system. Work and Scratch are NOT backed up. Files lost there cannot be recovered. Always copy important results back to MyData or MyProjects after jobs complete.

### Paths across environments

The same storage area appears at different paths depending on where it is accessed.

JupyterHub paths (mounted in the notebook file browser)

| Storage Area | Path |
|---|---|
| MyData | `/home/jupyter/MyData/` |
| MyProjects | `/home/jupyter/MyProjects/PRJ-...` |
| Work | `/home/jupyter/Work/stampede3/` |
| CommunityData | `/home/jupyter/CommunityData/` |
| NHERI Published | `/home/jupyter/NHERI-Published/PRJ-...` |

Stampede3 paths (absolute UNIX paths on the HPC system)

| Storage Area | Path |
|---|---|
| Home | `/home1/<groupid>/<username>/` |
| Work | `/work2/<groupid>/<username>/stampede3/` |
| Scratch | `/scratch/<groupid>/<username>/` |

Actual paths on Stampede3 can be verified with `echo $HOME`, `echo $WORK`, and `echo $SCRATCH`.

### How dapi translates paths

When submitting jobs through dapi, a single call converts DesignSafe paths to Tapis URIs.

```python
ds.files.to_uri("/MyData/analysis/")
```

This converts a familiar DesignSafe path into the Tapis URI that the job submission system requires. See the [dapi documentation](https://designsafe-ci.github.io/dapi/) for the full file-management API.

### File movement during job execution

Data moves through a predictable cycle during job execution.

1. Prepare input files in MyData or MyProjects.
2. Submit the job. Tapis stages inputs from DesignSafe storage to the execution system's Work or Scratch directory.
3. The application reads from and writes to local high-performance storage during execution.
4. After the job completes, Tapis archives results back to DesignSafe storage.

There is no need to manually transfer files between DesignSafe and TACC systems. Tapis handles staging and archiving automatically.

### File transfer tips

Many small files transfer slowly. A directory with 1,000 small CSV files takes significantly longer to stage than a single tar.gz archive containing the same data. Bundle input files before staging whenever possible to reduce overhead.

If multiple jobs share the same input data (for example, 500 ground-motion records reused across a fragility study), keep that data in Work to avoid re-staging it for every submission.

Archive only final outputs. Intermediate files and logs can remain in Work or Scratch temporarily, then be discarded once the important results have been copied.

Large datasets benefit from the higher I/O bandwidth of Work and Scratch. Running jobs directly against Corral (MyData) is slower and not recommended for production simulations.

## Next steps

- [Running HPC Jobs](job-resources.md) for submitting jobs, choosing resources, and understanding how jobs flow through the system
- [Debugging Failed Jobs](debugging.md) for reading output files and interpreting job states
- [Parallel Computing](parallel-computing.md) for MPI and multi-node execution
- [Parameter Sweeps](parameter-sweeps.md) for high-throughput studies with PyLauncher
- [DesignSafe Applications](../apps/overview.md) for the full catalog of available tools
