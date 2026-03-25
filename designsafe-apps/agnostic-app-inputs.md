# AgnosticApp - Input Arguments

*Input parameters and what each one means*

This section is the user manual for every input you see in the portal.

## 1. File input

### Input Directory (required)

A single directory staged by Tapis into the job.

What should be inside

- your main script (Tcl or Python)
- any supporting files your script needs (models, data, configs)
- optional helper files
  - *modules.txt* (for *MODULE_LOADS_FILE*)
  - *requirements.txt* (for *PIP_INSTALLS_FILE*)
  - *prehook.sh* / *posthook.sh* (for hook variables)
  - zipped bundles referenced by *UNZIP_FILES_LIST*

The wrapper *cd*s into this directory before running the main command. Relative paths in your script should assume this directory is the working directory.

---

## 2. Required app arguments

### Main Program (required)

The executable to run (binary name).

Common values

- *OpenSees* (serial Tcl)
- *OpenSeesMP* / *OpenSeesSP* (MPI Tcl)
- *python3* (Python workflows, including OpenSeesPy)

The executable must come from one of these sources

- available via modules (recommended), or
- present in the working directory / PATH

If Main Program is *python* or *python3*, the wrapper normalizes to *python3*.

---

### Main Script (required)

The filename of the input script passed to the executable.

Rules

- filename only (no path)
- must exist inside the Input Directory

Examples

- *model.tcl*
- *run_analysis.py*
- *Ex1a.Canti2D.Push.argv.tacc.py*

---

### UseMPI (required)
Controls whether the wrapper launches the executable through *ibrun*.

| UseMPI value | What runs |
|---|---|
| *False* | *<Main Program> <Main Script> [args...]* |
| *True*  | *ibrun <Main Program> <Main Script> [args...]* |

Use *True* when

- OpenSeesMP / OpenSeesSP
- Python + *mpi4py*

Use *False* when

- serial OpenSees (Tcl)
- serial Python / OpenSeesPy
- Python using threading / *concurrent.futures* within a node

> Note: the wrapper treats many "true-like" values as True (*True*, *1*, *Yes*, case-insensitive).

---

### CommandLine Arguments (optional)
Free-form arguments appended after the Main Script.

Example
```text
--NodalMass 4.19 --outDir outCase1
```

Final command structure
```bash
[ibrun] <MainProgram> <MainScript> <Arguments...>
```

---

## 3. Scheduler inputs

### TACC Scheduler Profile (defaulted)
The app uses the *tacc-no-modules* profile by default so no modules are implicitly loaded.
This is intentional. Module state is controlled explicitly by the wrapper to improve reproducibility.

### TACC Reservation (optional)
Provide a reservation string if you have one.

---

## 4. Environment variables (advanced configuration)

These values are presented as app inputs in the portal. Most are optional. If you never set them, the wrapper runs with conservative defaults.

### 4.1 OpenSeesPy injection

#### GET_TACC_OPENSEESPY (default *True*)
If True-like, the wrapper attempts to use the TACC-compiled OpenSeesPy by

- loading *python/3.12.11*, *hdf5/1.14.4*, *opensees*
- copying *${TACC_OPENSEES_BIN}/OpenSeesPy.so* into the working directory as *./opensees.so*

Use this when you want reliable OpenSeesPy on Stampede3 (recommended).

In your Python script

```python
import opensees as ops
```

If *TACC_OPENSEES_BIN* is unset or *OpenSeesPy.so* is missing, the wrapper logs a warning and skips the copy.

---

### 4.2 Module loading (two mechanisms)
The two mechanisms are complementary. You can use both.

#### A. MODULE_LOADS_FILE (optional)
A filename (in the Input Directory) containing module commands, one per line.

Supported line formats

- *purge*
- *use <path>*
- *load <module>*
- *?module* (optional *try-load*)
- bare module names

This is best for version-controlled, documented module stacks. It also makes submittal via the web-portal interface easier.

#### B. MODULE_LOADS_LIST (optional)
Comma-separated list of modules to load, e.g.
```text
python/3.12.11,opensees,hdf5/1.14.4,pylauncher
```

Tip: use *MODULE_LOADS_FILE* when the setup is more than a few modules or needs comments.

---

### 4.3 Python package installs (two mechanisms)
The two mechanisms are complementary. You can use both.

#### A. PIP_INSTALLS_FILE (optional)
A requirements-style file (in the Input Directory), e.g. *requirements.txt*.

Wrapper behavior

- runs *pip3 install -r <file>*
- fails the job if pip fails (with a clear error)

It makes submittal via the web-portal interface easier.

#### B. PIP_INSTALLS_LIST (optional)
Comma-separated list of packages, e.g.
```text
mpi4py,pandas,numpy,matplotlib
```

Wrapper behavior

- installs each package with *pip3 install <pkg>*
- fails the job if any install fails

---

### 4.4 Input preparation

#### A. UNZIP_FILES_LIST (optional)
Comma-separated list of ZIP files in the Input Directory to expand before execution.
Entries may omit the *.zip* suffix.

Use this when you staged one bundled zip instead of many small files.

#### B. PATH_COPY_IN_LIST (optional)
Comma-separated list of absolute paths (within the execution system) to copy into the working directory before execution.

Example
```text
$WORK/FileSet2,$SCRATCH/FileSet3/thisFile.at2
```

Use this when

- you need large or shared datasets without duplicating them into the Input Directory
- you want a specific runtime layout inside the working directory

#### C. DELETE_COPIED_IN_ON_EXIT (default *0*)
If set to *1* or True-like, the wrapper deletes only the copied-in items listed in its manifest on exit.

Safety rules

- refuses absolute paths
- refuses *..* traversal
- deletes only what landed in the working directory

Use this when copy-in files are temporary conveniences and should not be archived.

---

### 4.5 Pre/Post hooks

#### A. PRE_JOB_SCRIPT (optional)
Script to run after environment setup but before the main executable.
- if relative, interpreted as *./script* inside the Input Directory
- if executable, run directly; otherwise run via *bash*

#### B. POST_JOB_SCRIPT (optional)
Script to run after the main executable (same resolution rules as pre-hook).

Default policy: hook failures are logged as warnings and the job continues (you can change this policy in the wrapper if desired).

---

### 4.6 Output management

#### A. ZIP_OUTPUT_SWITCH (default *False*)
If True-like, the wrapper

- zips the entire Input Directory after execution into *inputDirectory.zip*
- removes the original directory

Use this when output is large and contains many small files, or when you want a single artifact to move or download.

#### B. PATH_MOVE_OUTPUT (optional)
If set, the wrapper moves the main output artifact into
```text
<PATH_MOVE_OUTPUT>/_<JobUUID>/
```
and copies top-level logs into that same folder.

Recommended

- move to *$WORK/...* for interactive inspection in JupyterHub
- move to *$SCRATCH/...* for chained HPC workflows


---

## 5. Typical patterns

### Serial OpenSees (Tcl)
- Main Program: *OpenSees*
- UseMPI: *False*

### OpenSeesMP / OpenSeesSP (MPI)
- Main Program: *OpenSeesMP* (or *OpenSeesSP*)
- UseMPI: *True*

### OpenSeesPy (serial)
- Main Program: *python3*
- UseMPI: *False*
- *GET_TACC_OPENSEESPY=True*

### Python + mpi4py
- Main Program: *python3*
- UseMPI: *True*
- *PIP_INSTALLS_LIST=mpi4py* (or requirements file)


#  AgnosticApp - How to Choose Inputs

*A one-page cheat sheet for designsafe-agnostic-app*

This app gives you a lot of power. You do not need to use most inputs for most jobs.

Use this page to answer four questions, top to bottom.

---

## 1. What am I actually running?

Pick one executable and one main script.

| If you are running | Set *BINARYNAME* | Set *INPUTSCRIPT* |
| --- | --- | --- |
| OpenSees (Tcl) | *OpenSees* | *model.tcl* |
| OpenSeesMP / OpenSeesSP | *OpenSeesMP* or *OpenSeesSP* | *model.tcl* |
| OpenSeesPy | *python3* | *run.py* |
| General Python | *python3* | *script.py* |
| Other executable | *<binary name>* | *script* |

Rules of thumb

* *INPUTSCRIPT* is just the filename, not a path
* The script must live inside *inputDirectory*
* For OpenSeesPy, always use *python3*

---

## 2. Do I need MPI?

Answer this explicitly. The app will not guess.

| If your job | Set *UseMPI* |
| --- | --- |
| Uses OpenSeesMP / OpenSeesSP | *True* |
| Uses *mpi4py* | *True* |
| Is fully serial | *False* |
| Uses Python threading or *concurrent.futures* on one node | *False* |

Mental model

```text
UseMPI = True   →  ibrun <command>
UseMPI = False  →  <command>
```

If you are unsure, start with *False* and turn it on only when needed.

---

## 3. Does my environment need anything special?

### A. Do I need modules?

If your code needs

* OpenSees
* MPI-aware HDF5
* a specific compiler stack

then yes, you need modules.

Choose one

| Use this when | Input |
| --- | --- |
| You want a clean, documented stack | *MODULE_LOADS_FILE* |
| You just need a few modules quickly | *MODULE_LOADS_LIST* |

Example mental model

```text
If I would type "module load …" on the command line,
I probably need to declare it here.
```

---

### B. Do I need Python packages?

| If you | Use |
| --- | --- |
| Have a *requirements.txt* | *PIP_INSTALLS_FILE* |
| Just need a couple of packages | *PIP_INSTALLS_LIST* |
| Need none | (leave blank) |

Packages are installed inside the job only.

---

### C. Am I using OpenSeesPy?

Almost always set this to True on Stampede3.

```text
GET_TACC_OPENSEESPY = True
```

Why

* Uses the TACC-compiled OpenSeesPy.so
* Avoids broken or incompatible PyPI wheels

---

## 4 How are my inputs structured?

### A. Is everything already in my input directory?

If yes

* do nothing
* simplest and safest case

---

### B. Do I have ZIP bundles?

Use when

* many small files
* datasets packaged for upload

Set

```text
UNZIP_FILES_LIST = mydata,meshes
```

(*.zip* is optional)

---

### C. Do I need large external data?

Use when

* data already exists in WORK / SCRATCH / HOME
* you do not want to re-upload it

Set

```text
PATH_COPY_IN_LIST = /work/.../dataset1,/scratch/.../dataset2
```

If the copy is temporary, also set

```text
DELETE_COPIED_IN_ON_EXIT = True
```

---

## 5. Do I need pre- or post-processing?

| Need | Input |
| --- | --- |
| Generate inputs | *PRE_JOB_SCRIPT* |
| Post-process results | *POST_JOB_SCRIPT* |
| Organize outputs | *POST_JOB_SCRIPT* |
| Cleanup | *POST_JOB_SCRIPT* |

Hooks

* run inside *inputDirectory*
* may be executable or run via *bash*
* failures warn by default (job continues)

---

## 6. Will my output be large or long-lived?

### A. Many output files?

Enable ZIP repack

```text
ZIP_OUTPUT_SWITCH = True
```

---

### B. Do I want results in WORK or SCRATCH immediately?

Move outputs

```text
PATH_MOVE_OUTPUT = $WORK/MyRuns
```

Results land in

```text
$WORK/MyRuns/_<JobUUID>/
```

This is highly recommended for

* large runs
* chained workflows
* interactive inspection in JupyterHub

---

## 7. Minimal starter configurations

### Serial OpenSees (Tcl)

```text
BINARYNAME   = OpenSees
INPUTSCRIPT = model.tcl
UseMPI      = False
```

---

### OpenSeesMP (MPI)

```text
BINARYNAME   = OpenSeesMP
INPUTSCRIPT = model.tcl
UseMPI      = True
```

---

### OpenSeesPy (recommended default)

```text
BINARYNAME            = python3
INPUTSCRIPT          = run.py
UseMPI               = False
GET_TACC_OPENSEESPY  = True
```

---

### Python + mpi4py

```text
BINARYNAME   = python3
INPUTSCRIPT = run.py
UseMPI      = True
PIP_INSTALLS_LIST = mpi4py
```

---

## 8. Final sanity checklist

Before submitting, ask yourself

*  Is my script inside *inputDirectory*?
*  Does *UseMPI* match how I actually run the code?
*  Do my modules match my binary?
*  Am I relying on any absolute paths unintentionally?
*  Do I want outputs zipped or moved?

If yes, you are good.

---

## 9. The core idea

The agnostic app is a general execution driver. Choose the simplest inputs that describe how you would run the job manually. The app makes that behavior explicit, safe, and reproducible.

# Run DS Agnostic App

*Notebook Demo for Submitting General HPC Jobs with the Agnostic App*

This notebook demonstrates how to submit computational jobs on DesignSafe using the `designsafe-agnostic-app`, a general-purpose Tapis application for running arbitrary workloads on HPC systems.

The focus here is not on a specific scientific domain, but on how jobs are constructed, submitted, and executed on DesignSafe. If you understand these notebooks, you understand the core execution model behind most automated workflows on the platform.

This demo intentionally uses a pure Python workflow to emphasize that the agnostic app is not tied to any particular software stack.

A similar notebook for OpenSees jobs is shown in the OpenSees-On-DesignSafe Document.

---

### Purpose of This Demo

This notebook showcases a general Python workflow that

* does not rely on OpenSees or any domain-specific tools
* runs exactly as it would on the command line
* uses the same submission mechanics as more complex HPC jobs
* demonstrates how DesignSafe handles execution context, environment setup, input/output staging, and reproducibility

The goal is to show that if you can describe a job in terms of

```
<executable> <script> <arguments>
```

then you can run it through the agnostic app.

---

### What to Watch For

As you work through the notebook, pay attention to these recurring patterns.

* The input directory defines the execution context
* The job command mirrors a standard command-line invocation
* MPI usage (or lack thereof) is explicitly controlled
* Software environments are declared, not assumed
* Outputs are organized to support large result sets, automation, and downstream workflows

These patterns apply broadly across DesignSafe, regardless of the application or discipline.

---

### What This Demo Covers

This notebook is

* a practical, end-to-end job submission example
* a reusable template for general workflows
* representative of how most Tapis jobs are constructed

It does not cover

* machine-learning tutorials
* performance optimization
* domain-specific training material

The emphasis is on workflow mechanics, not scientific content.

---

The agnostic app is a general execution driver. If you can run a job from the command line, you can run it on DesignSafe using this pattern.
