# OpenSees

Running [OpenSees](https://opensees.berkeley.edu/) on an HPC system like [Stampede3](https://www.tacc.utexas.edu/systems/stampede3) is not as simple as typing a command. Normally, you would need to

* Write a [SLURM](https://slurm.schedmd.com/) batch script, specifying resources, module loads, and MPI/OpenMP directives.
* Stage input files by manually copying them from your storage area to the cluster's scratch space for faster execution.
* Request the correct cores and nodes, balancing efficiency against allocation limits.
* Retrieve results by copying output files back from scratch to long-term storage.
* Troubleshoot environment setup, ensuring that OpenSeesMP or OpenSeesSP is properly compiled, loaded, and on your path.

These steps are essential for using HPC resources effectively, but they can also be error-prone and require deep familiarity with Linux, SLURM, and system architecture.

## OpenSees Tapis Apps

To simplify this process, DesignSafe provides three official OpenSees [Tapis](https://tapis.io/) Apps that automate job setup, submission, and file handling.

* OpenSeesEXPRESS, for sequential (single-core) simulations
* OpenSeesMP, for parallel simulations across multiple cores and nodes
* OpenSeesSP, for distributed-processing simulations across multiple cores and nodes
* OpenSeesPy, for parallel simulations across multiple cores and nodes (new)


## Where They Run

The *OpenSeesPy*, *OpenSeesMP* and *OpenSeesSP* apps run on Stampede3, where jobs are submitted through the SLURM scheduler and enter the HPC queue.

The *OpenSeesEXPRESS* app, by contrast, runs single-processor OpenSees on a Virtual Machine. These jobs do not go through the HPC queue, so they typically start immediately.



### The OpenSees Web-Portal app on DesignSafe automates this for you

The OpenSees Web-Portal app on DesignSafe does all of this heavy lifting. It

* generates the SLURM job script for you, tuned to the TACC environment (whether for *OpenSees*, *OpenSeesMP*, or *OpenSeesSP*),
* automatically stages your input files to the HPC scratch directory, where I/O is fastest,
* executes your analysis on the compute nodes you requested, and
* collects and returns your output files to your DesignSafe My Data workspace after the job completes.



This means you can

* focus on your structural or geotechnical model, not on cluster commands,
* submit jobs through a simple browser interface (or later, programmatically through Tapis or Python), and
* get consistent, optimized performance on TACC hardware without fighting with compilers, environment modules, or manual SLURM scripts.


## Why use the OpenSees app?

| Without the app | With the OpenSees app on DesignSafe |
| --- | --- |
| Write SLURM scripts by hand | SLURM generated automatically |
| Manually copy files to scratch | Inputs staged for you |
| Worry about correct module loads | Environment pre-configured |
| Track output files manually | Results copied back to My Data automatically |
| Manage tight coupling of resources | App sets up MPI/threads as needed |

The app automates this entire process so you can focus on your engineering analysis, not on cluster logistics.

## Where to find the DesignSafe app code

All of these applications are open source and maintained in the DesignSafe GitHub organization. The OpenSees app code lives at [WMA-Tapis-Templates](https://github.com/TACC/WMA-Tapis-Templates/tree/main/applications). There you will find several templates.

* The core TACC app JSON definitions (describing inputs, outputs, and parameters).
* JSON input schema files (describing what inputs the app expects).
* Docker or Singularity environments, if applicable.
* Examples of the underlying SLURM scripts it builds on your behalf, or submission logic.

This transparency lets you see exactly how your parameters translate into SLURM submissions, and how file staging is performed. Studying this repo is an excellent way to learn how your inputs turn into HPC jobs, which helps when you want to move to more advanced workflows (like building your own SLURM scripts or automating with Tapis).
