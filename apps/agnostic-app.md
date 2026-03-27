# designsafe-agnostic-app

*Agnostic Tapis App for General Execution, including Python, OpenSees, OpenSeesMP, OpenSeesSP, and OpenSeesPy*

| | |
| --- | --- |
| Platform | DesignSafe / TACC Stampede3 |
| Runtime | ZIP (HPC Batch) |
| Execution System | Stampede3 |
| Queue (default) | skx-dev |

## 1. Purpose and Design Philosophy

*designsafe-agnostic-app* is a general-purpose, HPC-oriented [Tapis](https://tapis.io/) application designed to support many computational workflows without baking assumptions into the app itself.

Instead of creating separate apps for

* OpenSees vs OpenSeesMP
* Tcl vs Python
* Serial vs MPI
* Small vs large output jobs

this app acts as a configurable execution driver.

All behavior is controlled by

* app inputs
* environment variables
* wrapper logic

This makes the app

* reusable
* transparent
* automatable
* easy to fork and extend

Tapis orchestrates the job.
[SLURM](https://slurm.schedmd.com/) executes it.
The wrapper enforces semantics and safety.

---

## 2. High-Level Execution Model

At runtime, the following happens.

1. Tapis stages *inputDirectory*
2. A SLURM batch job is submitted
3. *tapisjob_app.sh* executes on the first node
4. The wrapper

   * prepares the environment
   * stages inputs
   * selects MPI vs non-MPI execution
   * runs the main executable
   * manages outputs
   * produces structured logs

The minimum mental model for the app is

`[UseMPI?]  BINARYNAME  INPUTSCRIPT  ARGUMENTS`

<details>
  <summary><b>Detailed Execution Mode</b></summary>

1. Tapis stages your Input Directory to the job working directory.

2. SLURM starts the batch job on Stampede3.

3. *tapisjob_app.sh* runs on the first allocated node and

   - sets up summary and full environment logs

   - *cd*s into the Input Directory

   - prepares inputs (optional copy-in, optional unzip)

   - loads modules (optional file + optional list)

   - normalizes Python (*python* to *python3*)

   - installs Python packages (optional file + optional list)

   - optionally injects TACC-compiled OpenSeesPy (*opensees.so*)

   - optionally runs pre/post hooks

   - chooses MPI launcher (*ibrun*) or direct run

   - runs your executable + script + args

   - optionally zips output and/or moves results inside the exec system

   - records timers and exits with clear error handling


</details>




## 3. Logs you should look at first

Every job produces
- *SLURM-job-summary.log* (compact "what happened")
- *SLURM-full-environment.log* (full *env | sort* dump)

The summary log also records
- launcher decision
- module/pip actions
- timers (run-only and total)



---

## 4. Overview

*designsafe-agnostic-app* provides a single, well-instrumented execution interface for
- [OpenSees](https://opensees.berkeley.edu/) (Tcl), OpenSeesMP/SP (MPI), OpenSeesPy
- general Python workflows
- reusable HPC job patterns (copy-in, unzip, hooks, packaging, output movement)

It is designed to be debuggable, reproducible, and extensible, and to work as a template for future apps.


# AgnosticApp - Quick Reference

This table defines all user-facing inputs and environment variables supported by the app.
Details and relationships are explained in later sections.

### Core Execution Inputs

| Input | Required | Description |
| --- | --- | --- |
| inputDirectory | yes | Directory containing the main script and all runtime files |
| BINARYNAME | yes | Executable to run (*OpenSees*, *OpenSeesMP*, *OpenSeesSP*, *python3*, etc.) |
| INPUTSCRIPT | yes | Main script filename inside *inputDirectory* |
| UseMPI | yes | Whether to launch with *ibrun* (MPI) |

---

### Command-Line Arguments

| Input | Required | Description |
| --- | --- | --- |
| ARGUMENTS | no | Free-form command-line arguments appended after the script |

---

### Module and Environment Configuration

| Input | Required | Description |
| --- | --- | --- |
| MODULE_LOADS_LIST | no | Comma-separated list of modules to load |
| MODULE_LOADS_FILE | no | File listing modules to load |

---

### Python Package Management

| Input | Required | Description |
| --- | --- | --- |
| PIP_INSTALLS_LIST | no | Comma-separated list of pip packages to install |
| PIP_INSTALLS_FILE | no | *requirements.txt*-style file |

---

### OpenSees / OpenSeesPy Support

| Input | Required | Description |
| --- | --- | --- |
| GET_TACC_OPENSEESPY | no | Copy TACC-compiled *OpenSeesPy.so* into run directory |

---

### Input Staging and Preparation

| Input | Required | Description |
| --- | --- | --- |
| UNZIP_FILES_LIST | no | ZIP files in *inputDirectory* to expand |
| PATH_COPY_IN_LIST | no | Absolute paths (WORK/SCRATCH/HOME) to copy in |
| DELETE_COPIED_IN_ON_EXIT | no | Remove copied-in paths after job completes |

---

### Pre/Post Execution Hooks

| Input | Required | Description |
| --- | --- | --- |
| PRE_JOB_SCRIPT | no | Script to run before main executable |
| POST_JOB_SCRIPT | no | Script to run after main executable |

---

### Output Management

| Input | Required | Description |
| --- | --- | --- |
| ZIP_OUTPUT_SWITCH | no | Repack output directory into a ZIP |
| PATH_MOVE_OUTPUT | no | Move final output to WORK/SCRATCH/HOME |

For a detailed description of every input parameter, see [AgnosticApp Input Arguments](agnostic-app-inputs.md).
