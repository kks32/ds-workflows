# DesignSafe Tapis Apps

DesignSafe provides web-based simulation applications that connect domain-specific scientific software to high-performance computing (HPC) resources at TACC. You can model earthquake effects on structures, simulate turbulent fluid flow, or forecast coastal storm surge, all while running on TACC's Stampede3 system. You do not need to write your own SLURM scripts or manage job submissions manually.

This section covers three flagship apps.

* OpenSees, for structural and geotechnical engineering simulations
* OpenFOAM, for computational fluid dynamics (CFD)
* ADCIRC, for modeling coastal storm surge and hydrodynamics

These apps go beyond simple wrappers around command-line tools. They are curated HPC workflows that automate file staging, parallel execution, and result collection. Each app is constructed to match the typical needs of its user community, with differences in how it handles mesh decomposition, MPI job launch, or post-processing.

Students learning simulation, researchers submitting batch jobs, and engineers running calibrated regional models all benefit from the same accessible, transparent, and reproducible computing environment.

## Overview of DesignSafe Applications (Apps)

| App Name | Target HPC system | What you provide | What the app does for you | Where results go |
| --- | --- | --- | --- | --- |
| OpenSees (Tcl) | Stampede3 and other TACC clusters | Input script (*.tcl*) + supporting files | Generates SLURM job, stages files on scratch, executes analysis | Results copied to My Data |
| OpenFOAM | TACC clusters with CFD environments | Case directories (system/, constant/, etc.) | Prepares environment, runs solver, post-processes | Results to My Data |
| ADCIRC | TACC clusters with hydrodynamics stack | Forcing files, mesh, control inputs | Builds SLURM script, manages MPI, runs ADCIRC | Results to My Data |
| Custom post-processing | Interactive VM / Jupyter | Input data files | Runs Python/R/Matlab scripts on cluster or VM | Outputs in My Data or local notebooks |

## Build On What You Will Learn

This training module uses OpenSees as its primary example, but the principles extend to any scientific workflow on DesignSafe. The app framework shares a common architecture across all tools, with Tapis handling job submission, SLURM handling HPC scheduling, and the Web Portal providing user-friendly access. Understanding how these apps are structured and executed gives you general best practices for scientific computing on HPC systems.

If your project requires a simulation tool not yet available as a DesignSafe app, you can build your own. Many researchers and teams have done exactly that, often by starting with one of these three as a template. Choose the app that most closely matches your problem type or parallel model (for example, MPI-based for distributed jobs, thread-based for shared memory), then adapt it to your specific use case.

DesignSafe's open infrastructure gives you access to the app source code, SLURM templates, environment setups, and submission logic through the [TACC GitHub organization](https://github.com/TACC/WMA-Tapis-Templates). These examples are practical starting points for custom workflows, whether you are doing ensemble runs, sensitivity studies, or scaling up to multi-node simulations.

By learning how to use and understand these apps, you gain portable HPC skills that apply to other simulation domains and tools across many disciplines.

# Compare Apps

The following comparison highlights both the shared foundation and distinct execution logic of the three DesignSafe applications (OpenSees, OpenFOAM, and ADCIRC). Understanding these differences helps you choose the right tool, tailor your workflow, or extend the apps to meet advanced needs.


## What they have in common

All three apps make high-performance computing accessible from a simple web interface, so you can focus on your engineering or science problem rather than the logistics of running jobs on Stampede3. They all

* let you upload your inputs through the DesignSafe Web Portal,
* automatically generate the appropriate SLURM job scripts,
* stage your files to scratch on the HPC system for maximum I/O performance,
* execute your analysis, and
* bring the results back to your My Data workspace on DesignSafe.

They also all

* use Tapis under the hood for secure job submission and data transfers,
* rely on TACC's fair-share scheduler, and
* are open source, hosted in the DesignSafe GitHub organization.

## How they are implemented differently

| App | How it is structured under the hood | What it automates specifically |
| --- | --- | --- |
| OpenSees | Very lightweight JSON definitions that link to specific TACC-installed executables (*OpenSees*, *OpenSeesMP*, *OpenSeesSP*). Relies heavily on simple SLURM templates. | Generates a standard SLURM script and runs the correct binary. Copies input files to scratch, executes, and retrieves results. |
| OpenFOAM | Includes extra steps to handle decomposition (*decomposePar*) before the main solver and reconstruction (*reconstructPar*) afterward. The app JSON and SLURM templates explicitly incorporate these. | Automates multi-step CFD workflow by decomposing the mesh, running the parallel solver, then reconstructing results, all in scratch. |
| ADCIRC | Tailored SLURM scripts that emphasize large-scale MPI runs across many nodes. Often includes environment modules and scratch staging optimized for massive runs. | Sets up MPI execution across hundreds or thousands of cores, stages forcing and mesh files, and gathers huge result sets back to My Data. |

## Practical value for advanced users

Knowing these differences helps you

* Debug or extend workflows. If you want to move to direct SLURM submissions, you can look at the app's GitHub repo to see exactly what it submits.
* Customize for ensembles or parameter studies by borrowing the app's staging logic or SLURM structure.
* Understand performance trade-offs, like why OpenFOAM must do decomposition and reconstruction, or why ADCIRC scripts are designed for extremely large MPI runs.

## Quick comparison table

| App | Target problems | Parallel model used | Special stages | Typical scale |
| --- | --- | --- | --- | --- |
| OpenSees | Structural and geotechnical analysis | MPI (OpenSeesMP) or threads (OpenSeesSP) | None beyond scratch staging | From single core to hundreds |
| OpenFOAM | CFD (fluid flow, turbulence) | MPI | *decomposePar* before, *reconstructPar* after | Typically dozens to hundreds of cores |
| ADCIRC | Coastal surge and hydrodynamics | Large-scale MPI | Handles huge meshes and forcing files | Hundreds to thousands of cores |



### Next level: use the apps as your HPC recipe

Even if you ultimately move to more manual workflows (writing your own SLURM scripts for maximum control or using *t.jobs.submitJob* via Tapis), the DesignSafe app repos are an excellent starting point. They show

* the exact modules and environment settings required on TACC clusters,
* typical *srun* or *mpirun* patterns for large parallel jobs, and
* file staging best practices (why everything goes to scratch first, then returns).

You can use these apps as proven examples of best practices for each software stack, adapting their logic to scale up your work beyond what the simple Web Portal allows.
