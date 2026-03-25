# Tapis-App Glossary

This glossary is focused on [DesignSafe](https://www.designsafe-ci.org/) + [Stampede3](https://www.tacc.utexas.edu/systems/stampede3) workflows.

* **App**

    A JSON definition describing *what to run*, *how to run it*, and *what runtime environment is required*. It is a template for creating Jobs.


* **Job**

    An execution instance of an App with specified input files, parameters, and runtime arguments.
    A Job runs on the designated Execution System.


* **Execution System**

    A [Tapis](https://tapis.io/)-defined HPC or VM environment that runs jobs.
    Defines

    * Scheduler ([Slurm](https://slurm.schedmd.com/) for TACC HPC)
    * Login method (SSH)
    * Filesystem paths (*execDir*, *inputDir*, *outputDir*)
    * Allowed runtimes (ZIP, Singularity)
    * Default modules (if applicable)


* **runtime**

    Specifies how the application environment is provided.

    * ZIP means unpacked scripts/binaries run in the host environment
    * SINGULARITY means run inside a *.sif* container image
    * DOCKER is only supported on systems that allow Docker (not TACC HPC)

    Example
    ```json
    "runtime": "ZIP"
    ```

* **containerImage**

    The source of the app's runtime environment.

    * Path or URL to *.zip*/*.tgz* archive (ZIP runtime)
    * Path to *.sif* image (Singularity runtime)
    * Docker image reference (if DOCKER runtime allowed)

    Example (ZIP)
    ```json
    "containerImage": "tapis://designsafe.storage/apps/opensees-sp.zip"
    ```


* **jobType**

    Defines how the job is launched.

    * BATCH is submitted through Slurm, supports multi-node jobs, and is used for OpenSeesSP/MP and most HPC workflows
    * FORK runs directly on the host without a scheduler, for single-node, lightweight tasks

    Example
    ```json
    "jobType": "BATCH"
    ```


* **execSystemExecDir**

    Directory on the execution system where ZIP archives are unpacked, Singularity images may be copied, the job's Slurm launch script is placed, and the actual execution happens. This is the working directory for the running job.


* **execSystemInputDir**

    Directory where Tapis stages all job inputs.
    This is a shared filesystem on TACC, so all compute nodes can read the same staged files.


* **execSystemOutputDir**

    Directory where job output files are placed after completion.
    Tapis transfers files from here to the Job record's "outputs" listing.


* **Input Staging**

    The process by which Tapis

    1. Creates job directory on the execution system
    2. Copies all referenced input files into it
    3. Ensures shared visibility across compute nodes

    On TACC HPC, input staging goes to the same shared storage visible from all nodes.


* **Parameter**

    A primitive value passed into the app. Numbers, strings, flags, or options, used for command-line arguments, filenames, counts, etc.


* **ArgString**

    The final command that Tapis constructs using the executable, parameters, and input references.

    Example
    ```
    OpenSeesMP model.tcl -n 48 -param 3.2
    ```


* **Scheduler Options**

    Additional Slurm settings including queue/partition, number of nodes, cores per node, and wall time. These can be specified in the App and sometimes overridden in the Job.


* **Apptainer / Singularity**

    HPC container runtime used on TACC systems. Unprivileged (safe for multi-user clusters), executes *.sif* images, and binds host directories into the container. Used for reproducible, environment-controlled apps.


* **Modules (TACC Environment Modules)**

    System-provided software components that can be loaded at runtime.
    ```
    module load hdf5
    module load opensees
    module load python
    ```
    ZIP runtime apps rely heavily on modules.


* **Shared Filesystem**

    Parallel filesystem (e.g., */work2*, */scratch*) visible from all compute nodes.
    Ensures that multi-node jobs can access the same staged inputs without replication.
