# OpenFOAM

[OpenFOAM](https://www.openfoam.com/) is a powerful, open-source computational fluid dynamics (CFD) toolkit, widely used for modeling fluid flow, turbulence, heat transfer, and more. Running OpenFOAM on a large-scale HPC system like [Stampede3](https://www.tacc.utexas.edu/systems/stampede3) is notoriously complex for several reasons.

* You typically work with case directories containing *system/*, *constant/*, and *0/* folders, and these must be staged properly to the compute environment.
* Running in parallel requires careful setup with *decomposePar*, plus a correctly configured [SLURM](https://slurm.schedmd.com/) script that specifies the MPI processes and threads.
* After execution, you often need to reconstruct data (*reconstructPar*) and post-process, then gather results back into persistent storage.

Doing this manually means

* writing and testing your own SLURM batch scripts with the correct MPI parameters,
* loading specific TACC modules for OpenFOAM (which may vary by cluster and compiler stack),
* manually copying your case directory to the HPC's scratch space to ensure the solver can read and write efficiently, and
* cleaning up large intermediate files and pulling final results back to long-term storage.

## The OpenFOAM app on DesignSafe simplifies this entire workflow

The OpenFOAM Web-Portal app on DesignSafe takes care of these complexities by

* accepting your full OpenFOAM case directory, packaged for upload,
* automatically generating the SLURM submission script to run on TACC systems, with the correct *mpirun* or *srun* calls for your requested resources,
* staging your case files to the HPC scratch space for maximum I/O performance,
* executing your solver, then reconstructing the results (if required), and
* copying the results back to your DesignSafe workspace so you can download or continue post-processing.

This frees you to focus on building and validating your CFD model, not on the administrative overhead of HPC cluster workflows.

## Where to find the app's code

Like all DesignSafe apps, the OpenFOAM application is fully open source. You can explore the implementation at [WMA-Tapis-Templates](https://github.com/TACC/WMA-Tapis-Templates/tree/main/applications).

There you will find

* input parameter schemas (describing what files and options you can set),
* templates for SLURM submission scripts specific to OpenFOAM's decomposition and parallel execution, and
* any container definitions or environment module setups used to ensure compatibility across TACC systems.

Studying this repository is an excellent way to understand how your OpenFOAM inputs are translated into HPC jobs, which is especially valuable when you start writing your own automation or direct SLURM submissions.

## Comparison table

| Without the app | With the OpenFOAM app on DesignSafe |
| --- | --- |
| Manually prepare SLURM + MPI scripts | SLURM + MPI setup done automatically |
| Copy case files to scratch by hand | Input cases staged for you |
| Must load modules and ensure correct build | Environment pre-configured on TACC |
| Reconstruct + copy results back manually | Outputs automatically gathered into My Data |
| Handle parallel decomposition explicitly | Decomposition + execution managed by the app |

The app ensures your CFD analyses run efficiently on HPC without requiring you to become an expert in SLURM, MPI, or filesystem staging.
