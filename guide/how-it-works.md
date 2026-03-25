# Computational Workflows on DesignSafe

[DesignSafe](https://designsafe-ci.org) provides a comprehensive cyberinfrastructure for conducting, managing, and analyzing research workflows in natural hazards engineering. It brings together interactive computing environments, shared data services, and large-scale computational resources to support the full research lifecycle, from early model development to large ensemble simulations to advanced post-processing and data analysis.

At its core, DesignSafe is tightly integrated with the [Texas Advanced Computing Center (TACC)](https://www.tacc.utexas.edu/), which supplies the high-performance computing (HPC) systems and large-scale storage needed to execute demanding analyses. Most computational jobs launched through DesignSafe, whether from the web portal, a Jupyter notebook, or an automated pipeline, ultimately run on TACC systems such as [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/), enabling simulations and studies that would be impractical on a laptop.

The platform supports lightweight interactive exploration as well as production-scale simulations spanning many nodes and thousands of CPU cores. Data and compute are co-located at TACC, so earthquake ground-motion records, structural response datasets, and geotechnical field data hosted on DesignSafe can be accessed directly from the compute environment without downloading and re-uploading. A simulation that takes 12 hours on a laptop can finish in minutes by distributing the workload across many cores. Understanding how to move effectively between interactive and batch modes is essential for building efficient and reproducible research workflows.

## The DesignSafe Portal

Everything starts at the [DesignSafe web portal](https://www.designsafe-ci.org/rw/workspace/). The portal is the central hub for launching tools, managing data, and submitting jobs. From the portal workspace, researchers can

- Launch [JupyterHub](https://jupyter.designsafe-ci.org) for interactive notebooks (Python, R, Julia)
- Launch Jupyter HPC Native sessions on Stampede3 or Vista GPU nodes for heavier interactive work
- Open interactive desktop sessions via VNC or [DCV](https://docs.tacc.utexas.edu/tutorials/remotedesktopaccess/) for tools like [STKO](https://asdeasoft.net/stko/), [QGIS](https://qgis.org/), [ParaView](https://www.paraview.org/), and [VisIt](https://visit-dav.github.io/visit-website/)
- Submit HPC batch jobs for simulation applications ([OpenSees](https://opensees.berkeley.edu/), [OpenFOAM](https://www.openfoam.com/), [ADCIRC](https://adcirc.org/), [LS-DYNA](https://lsdyna.ansys.com/), MPM, and others)
- Access the [Data Depot](https://designsafe-ci.org/user-guide/datadepot/) to browse, upload, share, curate, and publish datasets

The portal's submission forms look interactive, but jobs submitted to HPC systems are batch jobs. The form simplifies the process of specifying inputs, resources, and parameters, but execution is queued, scheduled, and non-interactive.

Researchers can also bypass the portal entirely and submit jobs programmatically from a Jupyter notebook using [dapi](https://designsafe-ci.github.io/dapi/) or the [Tapis](https://tapis.readthedocs.io/en/latest/) Python client (`tapipy`). This is the preferred approach for automated pipelines, parameter sweeps, and reproducible workflows.

## Where computation happens

DesignSafe provides three types of compute environments, all accessible from the portal.

<img src="../images/compute-environments.svg" alt="DesignSafe compute environments overview" width="100%" />

**JupyterHub** runs on a [Kubernetes](https://kubernetes.io/) cluster at TACC. Each session gets a dedicated container with up to 8 CPU cores and 20 GB of RAM. Sessions start immediately with no queue. This is the primary environment for writing code, testing models, visualizing results, and submitting batch jobs to HPC. For work that needs more power than the standard JupyterHub container, Jupyter HPC Native sessions run directly on Stampede3 or Vista GPU nodes with access to full node resources, though these go through the SLURM queue.

**Virtual machines** at TACC run several applications that benefit from an interactive session without a queue wait. [MATLAB](https://www.mathworks.com/products/matlab.html), OpenSees Interactive, ADCIRC Interactive, STKO, and QGIS all run on shared VMs. Some (STKO, QGIS, ParaView) provide a full graphical desktop through VNC or DCV. VMs share hardware across users, so performance varies under load, and they are best for lightweight tasks and quick tests.

**HPC systems** at TACC handle production-scale computation. [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/), [Frontera](https://docs.tacc.utexas.edu/hpc/frontera/), and [Lonestar6](https://docs.tacc.utexas.edu/hpc/lonestar6/) are clusters of interconnected machines (nodes), each with dozens of CPU cores and hundreds of gigabytes of memory. A single job can span multiple nodes. HPC jobs wait in a queue managed by [SLURM](https://slurm.schedmd.com/documentation.html), which allocates hardware fairly across thousands of researchers. Long-running simulations, parallel analyses across many cores, and parametric sweeps with hundreds of runs all belong on HPC.

[Compute Environments](compute-environments.md) covers hardware specs, node types, and queue details for each system.

## Interactive and batch workflows

Computational research on DesignSafe falls into two modes of work.

Interactive workflows involve writing code, running it, inspecting results, modifying inputs, and iterating. The feedback loop is tight, often seconds between runs. JupyterHub, the VM-based sessions (MATLAB, OpenSees Interactive, QGIS), and desktop environments (STKO via VNC, ParaView via DCV) all support this mode.

Batch workflows involve submitting a job to a queue and collecting results later. There is no live interaction during execution. The job runs on dedicated hardware, often for minutes to hours, and Tapis archives the output when it finishes. All HPC jobs on Stampede3, Frontera, and Lonestar6 are batch jobs, whether submitted through the portal, dapi, or direct SSH.

Most research projects use both modes. A typical progression is to develop and test interactively in JupyterHub, then submit production runs as batch jobs to HPC.

## Choosing the right environment

| Situation | Environment | Example |
|---|---|---|
| Writing and testing code, visualizing results | JupyterHub (interactive) | Developing a Python post-processing script, plotting response spectra |
| Quick interactive session with a GUI tool | VM with VNC/DCV (interactive) | Exploring spatial data in QGIS, building an OpenSees model in STKO |
| Quick serial simulation test | VM (interactive or submit-only) | Testing an OpenSees Tcl model, running a short MATLAB analysis |
| Large or long-running simulation | HPC batch job | Nonlinear time-history analysis of a 3D building, ADCIRC storm-surge forecast |
| Hundreds of independent simulations | HPC batch with [PyLauncher](https://github.com/TACC/pylauncher) | Fragility study varying ground-motion intensity across 500 records |
| Parallel simulation needing many cores | HPC batch with [MPI](https://www.mpi-forum.org/) | Multi-node OpenFOAM CFD, ADCIRC mesh with millions of elements |
| GPU-accelerated computation or AI/ML | Jupyter HPC Native (Vista) or HPC GPU queue | Training a neural network, GPU-accelerated material simulation |

## Where data lives

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

When an HPC job is submitted through dapi or the portal, Tapis stages input files from DesignSafe storage to the execution system's Work filesystem before the job starts, and archives output back to DesignSafe storage after completion. This happens automatically.

[Storage and File Management](storage.md) goes deeper into paths, performance, and strategies for handling large file transfers.

## Next steps

- [Compute Environments](compute-environments.md) for hardware specs, node types, and environment selection
- [Storage and File Management](storage.md) for storage paths and file transfer tips
- [Running HPC Jobs](job-resources.md) for submitting jobs, choosing resources, and understanding how jobs flow through the system
- [Debugging Failed Jobs](debugging.md) for reading output files and interpreting job states
- [DesignSafe Applications](../apps/overview.md) for the full catalog of available tools
