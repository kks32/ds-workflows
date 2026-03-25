# Anatomy of a Tapis App

A [Tapis](https://tapis.io/) v3 App is built from a small set of core components that together define

* how the app is described and registered
* how the environment is prepared on the compute node
* how your scientific workflow is executed
* how input and output files are staged

Each component and its role in the runtime sequence is explained below.

---

:::{dropdown} 1. app.json (App Definition)

This is the primary Tapis application definition file. It contains all the top-level metadata and job configuration used by the Tapis Jobs API, including

* The app ID, version ("latest"), and description
* Runtime type and container image (ZIP, containerImage)
* Execution system and directory paths (e.g., *${JobWorkingDir}*)
* Job submission type and resource requirements (e.g., node count, cores per node, memory, wall time)
* Inputs and arguments required to run the application (input parameters and file requirements)
* Archive and output settings
* Tagging and user interface notes and options

This file is used by Tapis to register the app and to describe how it should be executed on the target HPC system.
In effect, *app.json* tells Tapis what this app is, how to run it, and what inputs/outputs to expect.


:::

:::{dropdown} 2. Scheduler Profile (System-Level Environment Initialization)

A scheduler profile configures the compute node before your app runs.

It determines

* The base environment (clean or preloaded)
* Whether the *module* command exists
* Default compiler/Python/MPI environment
* How [SLURM](https://slurm.schedmd.com/) launches the job

The apps in this notebook use

```
--tapis-profile tacc-no-modules
```

This ensures a clean environment so the wrapper script controls all modules.

Scheduler Profile vs envVariables

* A Scheduler Profile configures the system
* envVariables configure your app's behavior

:::

:::{dropdown} 3. ZIP Runtime Package

Each app is delivered as a single ZIP file stored in [DesignSafe](https://www.designsafe-ci.org/) storage.

```
designsafe.storage.default/.../appname/version/app.zip
```

At job runtime, Tapis

1. Copies the ZIP into the job working directory
2. Unpacks it
3. Executes the wrapper script inside

This ZIP functions like a simple container image.

:::

:::{dropdown} 3a. tapisjob_app.sh (Wrapper Script)
The wrapper script is contained in the ZIP Runtime Package.

This script contains all runtime logic.

* Logging and timers
* Normalizing python/python3
* Loading modules (OpenSees, python, HDF5, etc.)
* Installing Python packages
* Copying OpenSeesPy (*OpenSeesPy.so*) if requested
* Optionally unzipping inputs
* Choosing launch method (*ibrun* vs direct execution)
* Running the user's script
* Cleaning up
* Producing summary logs

This is the true entry point for the app.

:::

:::{dropdown} 4. ReadMe.md (App Documentation, Optional)

Although optional, including documentation improves

* transparency
* reproducibility
* ease of use in the DesignSafe portal

This file does not affect execution, but it is included in the ZIP bundle.

:::

:::{dropdown} 5. Input Directory (user provided)

Each app declares one required input directory.

This directory must contain

* The user's main script (Tcl or Python)
* Supporting files
* Data files
* Requirements or module files (if used)

Tapis stages this directory to the job working directory before execution, and the wrapper script uses it as the run directory.

:::

---

## Runtime Sequence Overview

```
app.json -> job submission -> Tapis validates job -> input staging -> unpack ZIP
-> execute wrapper script -> executes main program (launch user code) -> archive outputs


                      +------------------------+
                      |      Tapis App         |
                      |      (Registered)      |
                      +----------+-------------+
                                 |
          +----------------------+------------------------+
          |                      |                        |
   +------v-------+      +------v--------+      +--------v---------+
   |   app.json   |      | Scheduler     |      |  ZIP Package     |
   | (definition) |      |   Profile     |      |  (runtime image) |
   +--------------+      | (system init) |      +------+-----------+
          |              +------+---------+          +--+-----------+
          |                     |                    | tapisjob_app.sh
          |                     |                    | + README.md    |
          |                     |                    +----+----------+
          |                     |                         |
          v                     v                         v
  Defines inputs,     Loads system-level        Actual runtime logic:
  parameters, queue,  environment rules.        modules, pip, OpenSeesPy,
  environment, ZIP.                             launches the user script.

          +-------------------------------------------------------------+
          |                     Input Directory                          |
          | (user-provided scripts, models, data, ZIPs, req files)      |
          +-------------------------------------------------------------+

                              Runtime Flow
                              -----------
              app.json -> Tapis -> unpack ZIP -> run wrapper -> run script
```

# Custom Tapis Apps

On DesignSafe, you have two productive ways to run work on HPC systems.

* Use a public Tapis App. These are pre-configured, maintained templates for common tools (e.g., OpenSees, [OpenFOAM](https://www.openfoam.com/)). Fastest path to results, minimal setup.
* Author your own Tapis App. A custom template you control (wrapper + app.json, plus optional profile.json) when you need different binaries, launch logic, inputs/parameters, or project-specific defaults.

Start with a public app if your workflow fits its interface. Write a custom app when you need non-standard flags, containers/modules, pre/post steps, or a lab-specific interface you'll reuse.

---

## A. Use an Existing (Public) App

Public apps on DesignSafe are vetted templates for common tools. They're the fastest path to results.

Where to find them

* Web Portal, under Tools & Applications, to browse/search and read the app's help page.
* In notebooks/CLI, grab the app's *appId* (and optional version) from the portal and submit jobs programmatically.

How to run effectively

* Review the app's inputs and parameters (from its schema).
* Start with a small test case. Keep default resources, tune later.
* Prefer stable/latest versions shown in the catalog.
* Keep inputs on a Tapis-visible system (e.g., MyData, Work).

When public apps are ideal

* Your workflow matches the app interface.
* You want a supported, reproducible environment with minimal setup.
* You don't need custom launch logic or unusual dependencies.

---

## B. Write Your Own App

Create a Tapis App when you need custom behavior, different software versions, or a specialized interface for your project.

Building blocks

1. Wrapper (e.g., *tapisjob_app.sh*). Non-interactive, writes outputs to *$PWD*, handles launch (*ibrun*/*mpirun*/*srun*) and logging.
2. *app.json* (required). App ID/version, execution system/type (HPC), job type (MPI/SERIAL), defaults (nodes/ppn/walltime), inputs and parameters.
3. *profile.json* (optional). Modules and env vars (or use containers).
4. Tiny test dataset for validation.

Registration and sharing

* Register via portal or API, then share with your project/team.
* For catalog visibility, follow DesignSafe's review/publication process.

Best practices

* Version every change (e.g., *1.2.0*). Keep a changelog. Avoid breaking users.
* Use Tapis URIs and parameters. No hard-coded paths.
* Set sensible defaults (resources, inputs) and document them.
* Keep profiles minimal. Prefer containers or a short modules list.
* Provide clear labels/descriptions so users don't guess.

When custom apps shine

* New binaries/flags, custom pre/post steps, lab-specific UI.
* Automated parameter sweeps, ensembles, multi-stage workflows.
* A reproducible template your group can reuse.

---

## Quick Chooser

| Need | Public app | Your app |
| --------------------------------------- | :--------: | :------: |
| Fast start with a standard tool | Yes | |
| Custom binaries, flags, or launch logic | | Yes |
| Team-specific interface & defaults | | Yes |
| Minimal maintenance | Yes | |
| Full control / special dependencies | | Yes |

# Creating a Custom Tapis App

This is a comprehensive, unified guide. It walks you through creating a custom HPC-enabled Tapis app on DesignSafe that uses TACC's environment modules (not Docker), and is launchable through both code (Python + [Tapipy](https://github.com/tapis-project/tapipy)) and the DesignSafe GUI.

This guide ties together

* Tapis-based app registration and job management
* Module-based (non-container) execution on TACC resources
* DesignSafe-specific deployment and GUI launching

This guide helps you define a reusable, shareable app that

* Runs on TACC HPC systems like [Stampede3](https://www.tacc.utexas.edu/systems/stampede3) or [Frontera](https://www.tacc.utexas.edu/systems/frontera)
* Uses modules (like *module load python*, *OpenSees*, etc.)
* Can be launched from either the DesignSafe GUI or Tapipy in Python

---

## Prerequisites

* A [DesignSafe account](https://www.designsafe-ci.org)
* Basic knowledge of Python and shell scripting
* Access to a TACC execution system (e.g., *stampede3*)
* Installed tools (Python environment with *tapipy*, optionally *tapis-cli-ng* for terminal-based registration)

---

:::{dropdown} Step 1. Set Up Your App Directory

Structure your folder like this.

```
my-awesome-app/
├── run_analysis.py       # Your Python script
├── wrapper.sh            # Wrapper to load modules and run the script
├── app-definition.json   # Tapis app definition (for GUI + CLI)
```

:::

:::{dropdown} Step 2. Your Python Code (run_analysis.py)

Example script.

```python
import sys

if len(sys.argv) != 2:
    print("Usage: python run_analysis.py <input_file>")
    sys.exit(1)

with open(sys.argv[1], 'r') as f:
    content = f.read()

print("=== File Contents ===")
print(content)
```

:::

:::{dropdown} Step 3. Wrapper Script (wrapper.sh)

This script is executed on the HPC system.

```bash
#!/bin/bash
cd $WORK/$JOB_NAME

# Load the necessary environment modules
module load python/3.9

# Run your Python script using an input file provided by the user
python run_analysis.py "$input_file"
```

Make it executable.

```bash
chmod +x wrapper.sh
```

:::

:::{dropdown} Step 4. Tapis App Definition (app-definition.json)

```json
{
  "id": "my-awesome-app-1.0",
  "name": "My Awesome App",
  "version": "1.0",
  "executionSystem": "designsafe.community.execution",
  "deploymentPath": "apps/my-awesome-app/1.0",
  "templatePath": "wrapper.sh",
  "executionType": "HPC",
  "runtime": "LINUX",
  "jobType": "BATCH",
  "parallelism": "SERIAL",
  "maxJobs": 10,
  "defaultMemory": "2GB",
  "defaultProcessors": 1,
  "defaultNodes": 1,
  "defaultMaxRunTime": "00:30:00",
  "deploymentSystem": "designsafe.storage.default",
  "inputs": [
    {
      "id": "input_file",
      "details": {
        "label": "Input File",
        "description": "A file to process"
      },
      "required": true,
      "inputType": "URI"
    }
  ],
  "parameters": [],
  "outputs": [
    {
      "id": "stdout.txt",
      "value": {
        "default": "stdout.txt"
      }
    }
  ],
  "tags": ["custom", "python", "designsafe"]
}
```

:::

:::{dropdown} Step 5. Upload to DesignSafe/TACC

Upload your app files.

```bash
scp -r my-awesome-app/ yourusername@login.designsafe-ci.org:/work/apps/my-awesome-app/1.0
```

Or via the DesignSafe Data Depot, move the files to

```
/work/apps/my-awesome-app/1.0
```

:::

:::{dropdown} Step 6. Register the App via Tapis CLI (Optional)

If you prefer CLI.

```bash
pip install tapis-cli-ng
tapis auth login
tapis apps create -F app-definition.json
tapis apps list | grep my-awesome-app
```

:::

:::{dropdown} Step 7. Register and Use the App in Python with Tapipy

Authenticate and upload your files (same as above).

```python
from tapipy.tapis import Tapis
import json

client = Tapis(
    base_url="https://designsafe.dev.tapis.io",  # or production URL
    username="your-username",
    password="your-password",
    tenant_id="designsafe"
)
client.get_tokens()

# Register the app
with open("app-definition.json") as f:
    app_def = json.load(f)

client.apps.createAppVersion(body=app_def)
```

:::

:::{dropdown} Step 8. Upload an Input File

```python
with open("example_input.txt", "rb") as f:
    client.files.insert(
        systemId="designsafe.storage.default",
        path="input/example_input.txt",
        file=f
    )
```

:::

:::{dropdown} Step 9. Submit a Job from Python

```python
job = client.jobs.submitJob(body={
    "name": "my-first-designsafe-job",
    "appId": "my-awesome-app-1.0",
    "appVersion": "1.0",
    "inputs": {
        "input_file": "tapis://designsafe.storage.default/input/example_input.txt"
    },
    "archive": True,
    "archiveSystemId": "designsafe.storage.default",
    "archivePath": "archive/my-awesome-output"
})
print("Job submitted:", job.id)
```

:::

:::{dropdown} Step 10. Monitor and Download Output

```python
import time

while True:
    status = client.jobs.getJob(jobId=job.id).status
    print("Status:", status)
    if status in ["FINISHED", "FAILED", "CANCELLED"]:
        break
    time.sleep(10)

# Download output
client.jobs.getJobOutput(jobId=job.id, path="stdout.txt", destination="./stdout.txt")
```

:::

:::{dropdown} Step 11. Launch from DesignSafe GUI

Once registered

1. Go to Workspace, then Tools & Applications, then Private Apps
2. Click your app ("My Awesome App")
3. Fill in input fields (like file selectors)
4. Launch job. Tapis runs it on Stampede3 or Frontera
5. Results appear in your Project Data folder

:::

## Tips and Best Practices

* Use *parallelism: PARALLEL* for MPI or multithreaded jobs
* Use *$SCRATCH* for fast, temporary storage and *$WORK* for persistent storage
* Load any required tools (MATLAB, R, Python, etc.) with *module load* inside *wrapper.sh*
* Add *"helpURI"* to your app JSON for linking to documentation
