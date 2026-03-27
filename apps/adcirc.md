# ADCIRC

[ADCIRC](https://adcirc.org/) is a widely used high-performance model for simulating coastal storm surge, tides, and other hydrodynamic flows. It solves large-scale systems with complex meshes and often requires hundreds or thousands of cores to run efficiently.

Running ADCIRC on an HPC system like [Stampede3](https://www.tacc.utexas.edu/systems/stampede3) is challenging because

* you need to prepare multiple specialized input files, including meshes (e.g. *fort.14*), boundary conditions (*fort.15*), forcing data (*fort.61*...), and more,
* the solver requires careful MPI configuration, meaning you must write precise [SLURM](https://slurm.schedmd.com/) batch scripts that set up the parallel environment correctly,
* you have to manually stage your input files to the scratch space on the compute cluster (for high I/O speed), execute the solver, and then gather the output files, and
* if you want to run multiple storm scenarios (for example in an ensemble study), you must carefully manage directories, job submissions, and resource allocation to avoid collisions and maximize throughput.

## The ADCIRC app on DesignSafe automates this

The ADCIRC app on the DesignSafe Web Portal was created to eliminate these complexities. It provides

* a web-based interface to upload your ADCIRC input files, including your domain meshes, tidal and meteorological forcings, and control parameters,
* automatic generation of the SLURM submission script, correctly configured for your requested number of cores and wall time,
* automatic staging of your files to the compute cluster's scratch space, so that ADCIRC reads and writes at maximum speed,
* execution of the solver with MPI across your allocated nodes, and
* return of all result files to your DesignSafe workspace (My Data), organized for easy retrieval or further analysis.

This means you can run powerful storm surge or tidal simulations on thousands of cores, directly from a browser, without needing to write SLURM or MPI commands by hand.

## Where to find the app's code

Like all DesignSafe apps, the ADCIRC application is open source and hosted in the DesignSafe GitHub organization at [WMA-Tapis-Templates](https://github.com/TACC/WMA-Tapis-Templates/tree/main/applications).

The repository includes

* JSON schemas that define what input files and parameters the app expects,
* templates for SLURM scripts configured specifically for large-scale MPI runs of ADCIRC, and
* environment specifications to load the correct ADCIRC binaries and supporting libraries on TACC systems.

Reviewing this code is an excellent way to see exactly how your web inputs translate into HPC job submissions, which is very helpful when you want to transition to more advanced workflows (like running ensembles via custom automation).

## Comparison table

| Without the app | With the ADCIRC app on DesignSafe |
| --- | --- |
| Must write detailed SLURM + MPI scripts by hand | Submission script created for you automatically |
| Manually copy meshes and forcing files to scratch | Inputs staged to scratch for fast execution |
| Handle complex MPI decompositions directly | App manages MPI environment and process layout |
| Manually retrieve potentially huge result files | Outputs automatically copied back to My Data |

The app automates all these steps, letting you focus on building better surge models and analyzing impacts rather than on cluster mechanics.
