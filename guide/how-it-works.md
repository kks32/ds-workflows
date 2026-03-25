# Computational Workflows on DesignSafe

[DesignSafe](https://designsafe-ci.org) provides a comprehensive cyberinfrastructure for conducting, managing, and analyzing research workflows in natural hazards engineering. It brings together interactive computing environments, shared data services, and large-scale computational resources to support the full research lifecycle, from early model development to large ensemble simulations to advanced post-processing and data analysis.

At its core, DesignSafe is tightly integrated with the [Texas Advanced Computing Center (TACC)](https://www.tacc.utexas.edu/), which supplies the high-performance computing (HPC) systems and large-scale storage needed to execute demanding analyses. Most computational jobs launched through DesignSafe, whether from the web portal, a Jupyter notebook, or an automated pipeline, ultimately run on TACC systems such as [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/), enabling simulations and studies that would be impractical on a laptop.

The platform supports lightweight interactive exploration as well as production-scale simulations spanning many nodes and thousands of CPU cores. Data and compute are co-located at TACC, so earthquake ground-motion records, structural response datasets, and geotechnical field data hosted on DesignSafe can be accessed directly from the compute environment without downloading and re-uploading. A simulation that takes 12 hours on a laptop can finish in minutes by distributing the workload across many cores. Understanding how to move effectively between interactive and batch modes is essential for building efficient and reproducible research workflows.

<img src="../images/compute-environments.svg" alt="DesignSafe compute environments overview" width="100%" />

## Where computation happens

Most day-to-day work starts in [JupyterHub](https://jupyter.designsafe-ci.org), a browser-based environment running on a [Kubernetes](https://kubernetes.io/) cluster at TACC. Each session gets a dedicated container with up to 8 CPU cores and 20 GB of RAM, enough for writing code, testing small models, and visualizing results. Sessions start immediately with no queue. JupyterHub is also where researchers submit jobs to HPC systems using [dapi](https://designsafe-ci.github.io/dapi/) or the [Tapis](https://tapis.readthedocs.io/en/latest/) API, making it the central workspace for both interactive and batch workflows.

Jobs that need more power than JupyterHub can provide (long-running simulations, parallel analyses across many cores, or parametric sweeps with hundreds of runs) go to TACC's high-performance computing (HPC) systems: [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/), [Frontera](https://docs.tacc.utexas.edu/hpc/frontera/), or [Lonestar6](https://docs.tacc.utexas.edu/hpc/lonestar6/). These are clusters of interconnected machines (nodes), each with dozens of CPU cores and hundreds of gigabytes of memory. A single job can span multiple nodes when the problem is too large for one. HPC jobs wait in a queue managed by [SLURM](https://slurm.schedmd.com/documentation.html), which allocates hardware fairly across thousands of researchers.

Several applications ([MATLAB](https://www.mathworks.com/products/matlab.html), [OpenSees](https://opensees.berkeley.edu/), [ADCIRC](https://adcirc.org/), [STKO](https://asdeasoft.net/stko/), [QGIS](https://qgis.org/)) also run on shared virtual machines at TACC that start immediately without a queue, though they share hardware and are best for lightweight, short-running tasks.

## Interactive and batch workflows

Computational research on DesignSafe falls into two modes of work.

Interactive workflows involve writing code, running it, inspecting results, modifying inputs, and iterating. The feedback loop is tight, often seconds between runs. JupyterHub and the VM-based interactive sessions (MATLAB, OpenSees Interactive, QGIS) support this mode. Development, debugging, testing, visualization, and exploratory analysis all happen interactively.

Batch workflows involve submitting a job to a queue and collecting results later. There is no live interaction during execution. The job runs on dedicated hardware, often for minutes to hours, and Tapis archives the output when it finishes. All HPC jobs on Stampede3, Frontera, and Lonestar6 are batch jobs, whether submitted through dapi, the web portal, or direct SSH.

An important distinction: even when launched through the DesignSafe web portal's graphical interface, jobs submitted to HPC are batch jobs. The portal simplifies the submission form, but the execution model is queued, scheduled, and non-interactive.

Most research projects use both modes. A typical progression is to develop and test interactively in JupyterHub, then submit production runs as batch jobs to HPC.

## Choosing the right environment

| Situation | Environment | Example |
|---|---|---|
| Writing and testing code, visualizing results | JupyterHub (interactive) | Developing a Python post-processing script, plotting response spectra from an OpenSees run |
| Running a quick interactive session | Virtual Machine (interactive) | Testing an OpenSees Tcl model, running a short MATLAB analysis, exploring data in QGIS |
| Running a large or long simulation | HPC batch job | A nonlinear time-history analysis of a 3D building model, an ADCIRC storm-surge forecast |
| Running hundreds of independent simulations | HPC batch with [PyLauncher](https://github.com/TACC/pylauncher) | A fragility study varying ground-motion intensity across 500 records |
| Parallel simulation with domain decomposition | HPC batch with [MPI](https://www.mpi-forum.org/) | A multi-node OpenFOAM CFD simulation, an ADCIRC mesh with millions of elements |

[Compute Environments](compute-environments.md) covers each option in detail, including hardware specs and node types.

## Where data lives

Input files, simulation outputs, and shared datasets move between storage areas during a workflow.

| Storage area | What it holds | Backed up | Accessible from |
|---|---|---|---|
| MyData | Private files | Yes | JupyterHub, web portal |
| MyProjects | Shared project files | Yes | JupyterHub, web portal, collaborators |
| Work | Active HPC data | No | Compute nodes, JupyterHub |
| Scratch | Temporary fast storage | No (purged periodically) | Compute nodes |
| CommunityData | Public datasets | Yes | Everyone (read-only) |
| Published | Archived datasets with DOIs | Yes | Everyone (read-only) |

Because data and compute are co-located at TACC, simulation inputs stored in MyData or CommunityData can be referenced directly from HPC jobs without manual transfers. [Tapis](https://tapis.readthedocs.io/en/latest/) handles the file staging automatically when jobs are submitted through [dapi](https://designsafe-ci.github.io/dapi/) or the web portal.

[Storage and File Management](storage.md) goes deeper into paths, performance, and strategies for handling large file transfers.

## Next steps

- [Compute Environments](compute-environments.md) for hardware specs, node types, and environment selection
- [Storage and File Management](storage.md) for storage paths and file transfer tips
- [Running HPC Jobs](job-resources.md) for submitting jobs, choosing resources, and understanding how jobs flow through the system
- [Debugging Failed Jobs](debugging.md) for reading output files and interpreting job states
- [DesignSafe Applications](../apps/overview.md) for the full catalog of available tools
