# Tapis Apps

Tapis Applications are the foundation of the Tapis job workflow. They enable consistent, portable, and sharable execution of complex analyses across compute systems.

Tapis Apps are pre-configured software templates that let researchers run simulations, analyses, and data workflows on high-performance computing (HPC) systems without needing to write job scripts or manage SLURM directly. They encapsulate the executable, expected inputs/parameters, and the target execution system so you don't have to write or maintain scheduler scripts.

A Tapis App is a preconfigured, reusable execution template that defines

* The executable (binary program/script/container) to be run (e.g., OpenSees, OpenFOAM, custom scripts)
* What inputs, parameters, and environment variables it expects
* The execution system (like Stampede3) where the job should run
* The runtime environment which describes how the job should be launched (e.g., runtime settings, number of cores)

:::{card}
Apps decouple *what to run* from *where/how to run it*, so users don't need to write scheduler scripts or manage modules. Most importantly, they allow researchers to focus on science, not infrastructure.
:::


---
## Why Use Tapis Apps?

Tapis apps provide a consistent workflow across systems and schedulers, reproducible execution (with versions, defaults, and metadata tracked), and portability across the Web Portal, Python (Tapipy), and CLI.

| Benefit | What it does |
| ----------------------------- | -------------------------------------------------------------------------------------------- |
| Simplifies HPC access | No need to SSH, stage files manually, or write job scripts |
| Standardizes environments | Ensures the right modules, compilers, and binaries are used across all users |
| Optimizes performance | Automatically runs jobs from fast storage (e.g., *$SCRATCH*) for best I/O performance |
| Automates file handling | Input/output files are transferred cleanly between your DesignSafe workspace and HPC systems |
| Enables reproducibility | Job metadata and configuration are tracked and reproducible for later re-use |

Researchers can focus on engineering and science, while Tapis ensures jobs are run efficiently and consistently across HPC systems.


---
## Which Apps to Use

You can use registered Tapis Apps (e.g., OpenSees, OpenFOAM), or you can write your own using a registered app as a template.

The registered apps are available through the DesignSafe Web Portal, or programmatically via Jupyter notebooks or the Tapis CLI. Each app encapsulates all the information needed to run a specific scientific application using best practices for performance and reproducibility.

---

## What Do Tapis Apps Do?

When you submit a job via a Tapis App, the system automatically

* Generates a SLURM script tailored to the app
* Stages your input files to the HPC system (e.g. *$SCRATCH*)
* Submits the job to the correct queue on a system like Stampede3
* Runs the job with the right environment, modules, and executable
* Returns the output back to your DesignSafe workspace (My Data)

Tapis handles the technical complexity behind the scenes, including queueing, execution, module loading, and file movement, so you can focus on the science.

> Under the hood, your app submission becomes a SLURM batch job, executed on a system like Stampede3 or Frontera.


---
## Typical Inputs to a DesignSafe Tapis App

Whether submitted through the web portal or programmatically, most apps expect

* A main input file (e.g., *model.tcl* for OpenSees or *input.json* for a custom workflow)
* Supporting files (e.g., data sets, configuration files, libraries)
* Run parameters (e.g., number of cores, wall time, flags)
* Output preferences (optional controls over where and how results are returned)

Because the app handles the SLURM environment for you, you don't need to write a job script manually unless you want more advanced control.


#  Tapis App Types

Tapis Apps provide a reproducible, portable way to run scientific workflows on DesignSafe's HPC systems. Each app defines what to run, how to run it, and what environment it runs in, while Tapis handles staging, execution, monitoring, and output collection. This section summarizes the core concepts so users understand how an app translates into a running job on systems such as Stampede3, Frontera, and Jetstream2.

---

##  1. What a Tapis App Describes

A Tapis App is a JSON specification that declares

1. Runtime type, meaning how the application environment is provided (ZIP, Singularity/Apptainer, or Docker).
2. Execution mode, meaning whether the job runs through a scheduler (BATCH) or directly on the host (FORK).
3. Command and arguments, the executable or script to run once the environment is ready.
4. Inputs, parameters, and outputs, for file staging, primitive parameters, directory handling, and result collection.

The App does *not* run anything by itself. It defines the template for future Jobs.

---

##  2. The App-System-Job Relationship

Tapis execution involves three layers.

1. App (what to run) specifies runtime, executable path, input schema, environment variables, etc.

1. Execution System (where it runs) defines login protocol (SSH), scheduler (Slurm), queue options, filesystem locations for input/output/exec, and which runtime technologies are allowed (ZIP, Singularity). DesignSafe/TACC systems allow ZIP and Singularity, not Docker.

1. Job (an instance of the app) receives user-provided input values, stages them into the execution directory, and launches under the system's scheduler or host environment.

---

##  3. Runtime Types

Tapis supports three runtime types.
On DesignSafe HPC, ZIP and SINGULARITY are the standard choices.

---

:::{dropdown} A. ZIP Runtime (Lightweight, Module-Based)

The app's `containerImage` is a `.zip`/`.tar` archive containing scripts, templates, and wrapper files.
At job start

1. The archive is copied into the job's execution directory.
2. It is unpacked.
3. The designated executable is run directly on the host, using the system's modules (e.g., `module load hdf5`, `module load opensees`, MPI).

This runtime is best for

* Apps that rely on TACC modules
* OpenSees/OpenSeesPy workflows
* Rapid iteration without building containers
* Simple driver scripts, Tcl/Python wrappers

DesignSafe uses this heavily because it is fast, simple, and takes advantage of HPC-optimized software already provided by TACC.

:::

:::{dropdown} B. SINGULARITY / APPTAINER Runtime (Reproducible, Self-Contained)

Singularity (now Apptainer) is the HPC container engine used on all TACC clusters.

The app's `containerImage` refers to a `.sif` file containing a complete software environment.
At runtime, Tapis does something similar to

```bash
apptainer run myapp.sif [args]
```

All inputs and outputs are bound into the container, and your executable runs inside a fully reproducible user-space environment.

This runtime is best for

* Custom scientific environments
* Conda-based workflows
* Applications not available as TACC modules
* Cross-cluster portability (Jetstream2, Stampede3, local)
* Stable, version-locked pipelines

If your app needs a specific OS, Python stack, compiler, or third-party library, Singularity gives you full control.

:::

:::{dropdown} C. DOCKER Runtime (Cloud and VM Use Only)

Docker images are supported only when the execution system allows Docker.
HPC compute nodes at TACC do not support Docker, so this runtime is not used on DesignSafe systems.

Use Docker only for apps running on VMs (e.g., Jetstream2) where the system explicitly declares Docker support.
:::

---

##  4. Execution Model

:::{dropdown} A. BATCH Jobs

BATCH jobs are submitted to the system scheduler (Slurm for TACC). They support multi-node jobs and are required for MPI/OpenSeesMPI/OpenSeesMP. Outputs are collected after completion.

Most HPC workloads, including all OpenSeesMP apps, use BATCH.

:::

:::{dropdown} B. FORK Jobs

FORK jobs run directly on the login/exec host without a scheduler. They are single-node, short, lightweight tasks, good for quick utilities or file preprocessing.

FORK jobs mean

* No scheduler (no SLURM, no PBS)
* No queueing
* No resource allocation
* Tapis executes *tapisjob.sh* directly on the target host

This is appropriate for small utility servers, virtual machines (VMs), lightweight services, and systems without a batch scheduler.

On large HPC systems like Stampede3, FORK jobs are not used. All compute work must go through SLURM. Batch jobs are mandatory to enforce fair usage and resource limits.

DesignSafe apps typically avoid FORK except for trivial tools.
:::

---

##  5. Input Staging and Shared Filesystems

Regardless of runtime type, Tapis stages all inputs into a single job directory on a shared filesystem (e.g., `/scratch`, `/work2`).
Because TACC systems use shared parallel filesystems

* Every compute node can see the same staged files
* Inputs are not copied separately to each node
* Multi-node jobs simply access the same directory

This is crucial for MPI and OpenSeesMP workflows.

---

##  6. Job Launch Flow (What Actually Happens)

A Tapis job on DesignSafe follows this flow.

1. User submits job request with parameter values and input file references.
2. Tapis creates the job directory (`execSystemExecDir`) on Stampede3.
3. Input files are staged into that directory.
4. The App's runtime is applied (ZIP archives are unpacked, Singularity `.sif` files are mounted and evaluated).
5. Tapis constructs a launch script (Slurm batch script for BATCH jobs).
6. The job is submitted through Slurm.
7. The job runs on compute nodes.
8. Output files are collected into `execSystemOutputDir`.
9. Tapis returns job status and output metadata.

This model is consistent across all Tapis-supported DesignSafe HPC apps.

---

##  7. When to Use Each Runtime (Rule of Thumb)

| Use Case | Best Runtime |
| --------------------------------------------- | -------------------------------------------------------------------------------- |
| Relying on TACC modules (OpenSees, HDF5, MPI) | ZIP |
| Need reproducibility across clusters | SINGULARITY |
| Custom Python/Conda environments | SINGULARITY |
| Fast development & simple scripts | ZIP |
| VM-based workflows with Docker available | DOCKER |
| OpenSeesMP / multi-node jobs | ZIP or SINGULARITY (both work. ZIP is simpler if modules are sufficient) |

---

# What Is Apptainer?

Apptainer is a container runtime designed specifically for HPC (High-Performance Computing). It provides the same core functionality as Docker (packaging your software and environment into a portable, reproducible container) but it is built to work without root privileges, which is essential in shared supercomputing environments.

It is

* Open-source, governed by the Linux Foundation
* The successor to Singularity (the project was renamed starting in 2021)
* The standard on HPC systems, where Docker is typically prohibited

Most people in HPC still casually say "Singularity," but the modern software stack and project name is Apptainer.

---

## Why HPC Uses Apptainer Instead of Docker

Supercomputers do not allow root access or privileged daemons. Docker requires a root-owned daemon (`dockerd`) and privileged operations (namespaces, cgroups, mounts). This is unsafe on multi-user HPC clusters.

Apptainer, however

* Runs containers as the user, with the same permissions the user already has
* Does not require a daemon
* Plays nicely with job schedulers like Slurm, PBS, LSF, etc.
* Integrates with shared filesystems (Lustre, GPFS, NFS)

This makes Apptainer the standard container runtime for academic HPC.

---

## How an Apptainer Container Works

An Apptainer container is typically a single immutable `.sif` file.

```
mycontainer.sif
```

This file contains

* A full Linux filesystem (like an OS image)
* Installed software/packages
* Executables, libraries, Python environments, etc.

You run it like this

```bash
apptainer run mycontainer.sif
```

or

```bash
apptainer exec mycontainer.sif python script.py
```

It feels like Docker, but the security model and execution model are very different.

---

## Apptainer vs Docker (quick comparison)

| Feature | Docker | Apptainer |
| ------------------------- | -------------------- | ----------------------------- |
| Requires root? | Yes | No |
| Designed for HPC? | No | Yes |
| Runs under Slurm? | Not allowed | Standard |
| File is a single `.sif`? | No | Yes |
| Can import Docker images? | N/A | Yes, converts to `.sif` |
| User isolation | Strong | User-level, HPC-safe |
| Supports GPUs, MPI? | Yes (but not on HPC) | Yes, HPC-native |

Apptainer can pull and convert Docker images.

```bash
apptainer build my.sif docker://pytorch/pytorch
```

This makes it the bridge between Docker ecosystems and HPC ecosystems.

---

## How This Relates to Tapis Apps on DesignSafe

When your Tapis app specifies

```json
"runtime": "SINGULARITY"
```

it is essentially running through Apptainer on Stampede3 or Frontera.

The app's

```json
"containerImage": "/work/.../myapp.sif"
```

is the Apptainer/Singularity image file.

Inside the job, Tapis runs something like

```bash
apptainer run myapp.sif args...
```

(Older docs say "singularity run," but on TACC systems the backend is now Apptainer.)

---

## When You Should Use Apptainer in Your Apps

Use Apptainer (runtime = SINGULARITY) when

* You want a self-contained, portable environment
* You want to ensure the software stack is the same everywhere
* Your app needs custom Python/conda, exotic libraries, or OS-level dependencies
* You want to run the same container on Jetstream2, TACC, and another cluster

Do not use it if

* You want extremely fast iteration and don't want to rebuild images
* You're happy with TACC modules, in which case use ZIP runtime instead

---

## Definition for Reference

> Apptainer (formerly Singularity) is the container engine used on HPC systems because it runs securely without root privileges. It packages your application and its full software environment into a single portable `.sif` image and executes entirely within user space, making it the standard container runtime for Tapis apps on DesignSafe's HPC resources.

# Execution Flows

*(ASCII diagrams showing ZIP vs. Singularity execution flows)*

You can place these in your Tapis Apps Overview section as visual explanations.

---

## ZIP Runtime Execution Flow

```
          +---------------------------+
          |        User Job           |
          |   (inputs + parameters)   |
          +-----------+---------------+
                      |
                      v
            Tapis Stages Inputs
      (all files placed in job directory)
                      |
                      v
          +--------------------------+
          |  Copy ZIP archive into   |
          |  execSystemExecDir       |
          +-----------+--------------+
                      |
                      v
              Unpack ZIP Archive
    (scripts, wrappers, configs, run files)
                      |
                      v
      Load TACC Modules (e.g., OpenSeesMP,
         MPI, HDF5, Python, etc.)
                      |
                      v
         Tapis Constructs Launch Script
              (Slurm batch script)
                      |
                      v
             Slurm Schedules Job
                      |
                      v
      Host Executes Script Directly
 (no container; code runs inside HPC environment)
                      |
                      v
          Outputs Collected by Tapis
```

ZIP = "Unpack, use modules, run script directly on Stampede3/Frontera."

---

## Singularity / Apptainer Runtime Execution Flow

```
          +---------------------------+
          |        User Job           |
          |   (inputs + parameters)   |
          +-----------+---------------+
                      |
                      v
            Tapis Stages Inputs
      (all files placed in job directory)
                      |
                      v
     Singularity Image (.sif) Located
   (local path or copied into exec dir)
                      |
                      v
         Tapis Constructs Launch Script
        (typically includes a command like)
    ->  apptainer run image.sif <arguments>
                      |
                      v
             Slurm Schedules Job
                      |
                      v
   Singularity Container Starts on Node
   (binds input/output directories inside)
                      |
                      v
     Executable Runs *inside the container*
    (full environment defined by the .sif)
                      |
                      v
          Outputs Collected by Tapis
```

Singularity = "Start container, bind job dirs, execute inside container."

---

## ZIP vs. Singularity Side-by-Side

```
ZIP Runtime                          Singularity Runtime
--------------                      -----------------------
Unpack ZIP -> use system modules ->  Bind dirs -> start container ->
run script directly                  run executable inside image
```

# Run a Tapis App

A Tapis job submission has one lifecycle, but it can be described from two perspectives.

## A. What the App-User Does (the "Front" Process)
::::{card}
User-facing workflow, covering what you choose and what you provide (Portal / CLI / Tapipy)


:::{dropdown} Install Tapipy

Run this once to install the SDK.

```bash
pip install tapipy
```

tapipy may have already been installed in JupyterHub.
:::

:::{dropdown} Connect to Tapis

Create the client and log in.

```python
from tapipy.tapis import Tapis

# Replace with your credentials
t = Tapis(
    base_url="https://tacc.tapis.io",
    username="your-username",
    password="your-password",
    account_type="tacc"
)

t.get_tokens()  # Log in to Tapis
```

You only need to call *get_tokens()* once per session.

:::


:::{dropdown} 1) Choose an app (and version)

You specify an `appId` and version (e.g., `opensees-mp-s3` or `...-latest`).
This determines

* the input schema (what files/params are allowed)
* the runtime style (ZIP vs container)
* the wrapper entrypoint (what actually runs on the cluster)

:::

:::{dropdown} 2) Provide inputs and parameters

You provide

* file inputs (files/dirs that must be staged to the execution system)
* parameters (simple values like flags, paths, numeric settings)
* optional environment variables (modules, pip installs, custom toggles)
* Archive system ID (e.g. *"tacc-archive"*)
* Where you want your outputs to be stored (*archivePath*)

:::

:::{dropdown} 3) Define execution settings (job attributes)

You request compute resources.

* nodes, tasks/cores, walltime, queue/partition
* optional scheduler extras (reservation, constraints)

:::

:::{dropdown} 4) Submit Job

Tapis submits the job to the execution system and tracks status.

```python
job = t.jobs.submitJob(
    jobName="my-first-job",
    appId="hello-world-1.0",
    parameters={},      # Replace with actual app parameters if needed
    fileInputs=[],      # Or provide input files here
    archiveSystemId="tacc-archive",
    archivePath="myuser/outputs/hello-job",
    archiveOnAppError=True
)

print("Job submitted!")
print("Job ID:", job.id)
print("Status:", job.status)
```


:::

:::{dropdown} 5) Monitor execution

You can track status from the Portal, CLI, or API/Tapipy, plus logs produced by Slurm (stdout/stderr) and by your wrapper script (summary logs).


You can check on your job.

```python
job = t.jobs.getJob(jobUUID=job.id)
print("Current Status:", job.status)
```

Or just the status field directly.

```python
status = t.jobs.getJobStatus(jobUUID=job.id)
print(status.status)
```

Common job status values for filtering

* *PENDING*
* *STAGING_INPUTS*
* *RUNNING*
* *FINISHED*
* *FAILED*
* *CANCELLED*
* *PAUSED*
* *BLOCKED*

You can filter jobs by status.

```python
jobs = client.jobs.listJobs(status='FINISHED')
```

Or via search.

```python
search_query = json.dumps({"status": "FAILED"})
jobs = client.jobs.listJobs(search=search_query)
```


:::

:::{dropdown} 6) Retrieve outputs

When complete, outputs are available in the archive location and via the Files service for browsing/downloading/reuse.

* Outputs are archived to the configured archive system/path.
* You can browse, download, and reuse results in later workflows.

List available files.

```python
files = t.jobs.getJobOutputList(jobUUID=job.id)
for f in files:
    print(f.name, f.length)
```

Download a file.

```python
output = t.jobs.getJobOutputDownload(
    jobUUID=job.id,
    path="stdout.txt"
)

with open("stdout.txt", "wb") as f:
    f.write(output)
```

The file paths (like *"stdout.txt"*) depend on how your app writes output.


:::


:::{dropdown} Full Example Script (Submit, poll, list outputs)

```python
from tapipy.tapis import Tapis
import json

t = Tapis(
    base_url="https://tacc.tapis.io",
    username="your-username",
    password="your-password",
    account_type="tacc"
)
t.get_tokens()

job = t.jobs.submitJob(
    jobName="my-first-job",
    appId="hello-world-1.0",
    # parameterSet / fileInputs structure can vary by app definition
    parameters={},
    fileInputs=[],
    archiveSystemId="tacc-archive",
    archivePath="myuser/outputs/hello-job",
    archiveOnAppError=True
)

print("Job ID:", job.id)
print("Status:", job.status)

# Poll status
job2 = t.jobs.getJob(jobUUID=job.id)
print("Current Status:", job2.status)

# Filter jobs (example)
search_query = json.dumps({"status": "FAILED"})
failed_jobs = t.jobs.listJobs(search=search_query)
print("Failed jobs returned:", len(failed_jobs))
```
:::

::::

## B. What Tapis Does (the "Internal" Runtime Process)
::::{card}
Runtime workflow, covering what Tapis automates on the execution system (SSH + filesystem + scheduler + archiving).

This is the same lifecycle, described by the system actions that occur after you submit. The exact details vary by execution system and runtime type, but the pattern is consistent.

validate, stage, unpack/prepare, submit, monitor, archive


:::{dropdown} 1) Job Definition (Validation + job record creation)
A Tapis App is defined by

* app.json, with inputs, parameters, environment, runtime type
* tapisjob_app.sh, the wrapper script executed on the HPC system
* optional supporting files (profiles, modules, docs)

When you submit a job, Tapis

1. Validates your request against app.json (required inputs present, parameter types correct, enums/schema constraints satisfied, strictFileInputs enforced if enabled)
2. Creates a job UUID and stores the resolved configuration (effective values)

Only after validation does the job move into staging.

:::

:::{dropdown} 2) Staging inputs (file-transfer phase #1)

Tapis prepares the execution environment on the HPC system.

1. Creates a job working directory (location depends on the Execution System definition)
2. Stages your input directory/files into the job's working directory
3. Stages the runtime asset (ZIP bundle or container image reference)
4. Applies permissions and writes internal metadata for tracking

No execution occurs in staging. This phase is file preparation.

:::

:::{dropdown} 3) Runtime preparation (ZIP unpack / container plan)

For ZIP runtime, Tapis copies the ZIP into the job directory, extracts it in place, and makes tapisjob_app.sh executable. In a ZIP runtime, the extracted bundle is effectively your "app container," just implemented as a portable archive.

For container runtime (Singularity/Apptainer), Tapis ensures the image is available on the system, plans bind mounts (exec/input/output paths), and encodes the container command into the scheduler script. Tapis itself does not "run the container." The scheduler-run script does.

:::

:::{dropdown} 4) Scheduler submission

Tapis generates a batch script (e.g., Slurm), injects your resource requests and runtime command, then runs `sbatch`.
Tapis stores the scheduler job ID so it can poll state.

The batch script uses queue/partition, node/task/core counts, time limits, scheduler options (reservations, constraints), and execution-system profile settings.

:::

:::{dropdown} 5) Wrapper script execution (your code runs here)

This is where the app logic lives.

The app's tapisjob_app.sh typically

1. Initializes timers and logs
1. Loads modules (from defaults and/or user-provided lists)
1. Configures Python (optionally installs packages)
1. Chooses a launcher (MPI via ibrun / srun, or serial via direct execution)
1. Runs the main executable (OpenSeesMP, OpenSeesSP, OpenSees, python3, etc.)
1. Writes summary logs and exits with a code that Tapis can capture

Tapis does not interfere with what happens inside the wrapper. It only observes job state and outputs.

:::

:::{dropdown} 6) Archiving outputs (file-transfer phase #2)

After the app's wrapper exits

1. Tapis creates the archive directory on the archive system/path
1. Copies outputs (excluding anything filtered by archiveFilter)
1. Includes Slurm logs (stdout/stderr) and wrapper logs
1. If archiveOnAppError=true, it still archives even when the job fails

This archiving phase can be slow if the output contains many small files.

:::

:::{dropdown} 7) Completion and user visibility

Once archived, you can

* browse outputs in the portal or via the Files API
* download logs and results
* compare runs across job UUIDs
* rerun with modified parameters
* share results with collaborators (where supported)

This completes the Tapis lifecycle.

:::


::::

The central idea is that Tapis is an orchestrator. It stages files, generates a scheduler script, submits to Slurm, monitors, then archives outputs.

## The Lifecycle at a Glance (Swimlane)
:::{card}

```
USER (Portal / CLI / Tapipy)              TAPIS (Jobs Service + Files)                 HPC (Stampede3 / Slurm)
------------------------------------     -----------------------------------------   -------------------------
1) Pick app + version  ------------>     Validate request (app schema)
2) Provide inputs/params ---------->     Create job record (UUID, config)
3) Request resources -------------->     Stage inputs + runtime (file transfer)
                                         Build batch script
                                         sbatch batch_script ------------------->    Queue (PENDING)
                                         Poll scheduler status  <-----------------   Run (RUNNING)
4) Monitor status  <-----------------   Map scheduler states to Tapis states
5) Get results  <--------------------   Archive outputs (file transfer)
                                         Provide outputs via Files API
```
:::


> On shared systems like Stampede3, jobs may queue before running due to demand. This delay is the trade-off for accessing powerful resources.



## Appendix. Tapis Job Execution (SSH + Slurm Timeline)
:::{card}

Tapis does not run jobs internally. The Jobs service automates what you would otherwise do manually on an HPC system.

1. SSH into the execution system (as the effective HPC user)
1. Create job directories
1. Stage inputs and runtime assets
1. Write a scheduler batch script
1. Submit and monitor the scheduler job
1. Archive outputs and expose them via the Files service

Condensed behind-the-scenes timeline

    SSH -> mkdir job directories
    SSH + Files -> stage inputs
    SSH -> copy/unpack ZIP (or locate container image)
    SSH -> write scheduler script
    SSH -> submit (sbatch)
    SSH -> poll (squeue / sacct)
    SSH -> collect output metadata
    Files Service -> deliver outputs

:::

## Where to Look When Debugging

::::{card}

If stuck in STAGING_INPUTS, investigate input transfers, too many files, or remote source delays.

If stuck in QUEUED/PENDING, investigate scheduler wait time (partition, allocation, walltime).

If failing in RUNNING, investigate wrapper logic, module loads, env vars, or executable errors.

If slow after FINISHED, investigate archiving overhead (again, too many files).


:::{card}
Many "slow jobs" are not slow because compute is slow. They are slow because file transfer is slow. The input staging and output archiving phases can dominate runtime when there are many small files. When possible, reduce file counts, reuse common datasets from Work/Scratch, or bundle inputs/outputs as a ZIP/TAR that you unpack/pack inside your wrapper.
:::

---


When users say "Tapis is slow," it usually means one of these stages is the bottleneck.

* Slow before RUNNING means input staging or queue wait
* Slow after FINISHED means archiving (lots of files or large directories)
* Slow during RUNNING means your executable/runtime environment

File-transfer advice (high impact)

* Minimize file count (thousands of small files is worse than one big file)
* Keep common datasets (e.g., ground motions) in Work/Scratch, and reuse them
* Bundle inputs/outputs as ZIP/TAR and extract/pack inside the wrapper
* Consider writing intermediate results to Work/Scratch and collecting only what you need at the end

::::
