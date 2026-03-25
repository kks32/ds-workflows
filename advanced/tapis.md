# Tapis Internals

This page consolidates the full technical detail of how [Tapis](https://tapis.readthedocs.io/en/latest/) works behind the scenes. You do not need any of this to run jobs on DesignSafe, but the information here is valuable when debugging failures, developing custom apps, or profiling performance.

## What Tapis Is

Tapis is a network-based platform that automates job submission, monitoring, and data management across HPC and cloud systems. It sits between your working environment (Jupyter notebooks, Python scripts, or the DesignSafe portal) and the execution systems at [TACC](https://www.tacc.utexas.edu/). You describe *what* you want to run, and Tapis handles *how* and *where* it runs.

There are three ways to interact with Tapis.

| Approach | Description | Audience |
|---|---|---|
| [dapi](https://designsafe-ci.github.io/dapi/) | High-level Python wrapper that generates job requests from a few parameters | Researchers, students |
| [Tapipy](https://tapis.readthedocs.io/en/latest/technical/pythondev.html) SDK | Python SDK auto-generated from the Tapis OpenAPI spec, with 100% API coverage | Application developers |
| REST API | Raw HTTP interface (GET, POST, PUT) accessible from any language or curl | System integrators, non-Python clients |

dapi calls Tapipy under the hood. Tapipy calls the REST API under the hood. Each layer adds convenience without removing capability.

::::{dropdown} Tapipy SDK example

```python
from tapipy.tapis import Tapis

client = Tapis(base_url="https://tacc.tapis.io",
               username="your-username",
               password="your-password",
               account_type="tacc")

client.get_tokens()

job_request = {
    "name": "example-job",
    "appId": "hello-world-1.0",
    "archive": True,
    "archiveSystemId": "tacc-archive",
    "archivePath": "your-username/job-output"
}

job = client.jobs.submitJob(body=job_request)
print("Job ID:", job['id'])
```

::::

::::{dropdown} REST API example (curl)

```bash
curl -X GET https://tacc.tapis.io/v3/jobs \
  -H "Authorization: Bearer <token>"
```

::::

---

## Job Lifecycle States

A Tapis job is a single execution of a registered Tapis App with your inputs, parameters, and resource requests. Jobs are portable, asynchronous, and tracked through a lifecycle of well-defined states.

| State | Meaning |
|---|---|
| *PENDING* | The job has been submitted but has not started running yet |
| *STAGING_INPUTS* | Tapis is copying input files to the execution system |
| *QUEUED* | The job is in the HPC scheduler queue, waiting for resources |
| *RUNNING* | The job is actively executing on compute nodes |
| *ARCHIVING* | Tapis is saving output files to the archive system (e.g., Corral) |
| *FINISHED* | The job completed successfully and outputs were archived |
| *FAILED* | Something went wrong (bad input, runtime error, system issue) |
| *CANCELLED* | The job was manually cancelled before completion |
| *BLOCKED* / *PAUSED* | Execution is held up due to system policies or errors |

You can query the current state at any time with `getJob()` or `getJobStatus()`, filter jobs by state (e.g., show all `FAILED` jobs), trigger downstream steps when a job reaches `FINISHED`, and debug problems when a job ends in `FAILED` or never leaves `PENDING`.

### Portability

:::{dropdown} Jobs are portable across execution systems

When a job is described as portable, it means the job can be moved or rerun in a different environment without requiring major changes.

| Portability aspect | In Tapis Jobs |
|---|---|
| Not tied to a single machine | You can run the same job on different execution systems (e.g., Stampede3, Frontera) as long as the app is registered there |
| Encapsulated configuration | The job includes references to all the inputs, parameters, and resource requests it needs |
| Repeatable and reproducible | Because the job schema is structured and versioned, you can re-run it later (or elsewhere) with the same result |
| Remotely accessible | You do not need to be logged into a specific cluster. You can submit and manage jobs from anywhere via API |
| Scriptable and automatable | You can define and launch the job using code (e.g., Python, Bash, JSON) rather than manual setup on one system |

You define a job for an OpenSees simulation with input files on a Tapis-accessible system, parameters in JSON, a specific app version, and a target system set to *stampede3*. Later, you can change the system to *frontera* (if supported), reuse the same app and inputs, and submit the same job again without rewriting anything. The "what to run" is separated from the "where to run it."

:::

### Asynchronous Execution

:::{dropdown} Jobs run independently after submission

| Asynchronous aspect | In Tapis Jobs |
|---|---|
| You do not have to wait | When you submit a job, you immediately get a response (job ID), and your script or notebook can move on |
| The job runs in the background | Tapis handles the lifecycle (staging, running, archiving) without requiring you to stay connected |
| You can check on it later | Monitor status (*PENDING*, *RUNNING*, *FINISHED*, etc.) and fetch outputs when the job completes |
| Useful for large or long tasks | Asynchronous execution is ideal for simulations that take minutes, hours, or days to finish |
| Job state is managed by Tapis | Tapis maintains a full record of job metadata, status, inputs/outputs, and logs, independent of your session |

With asynchronous jobs, you can fire off a job from a notebook or script, continue working or log off, and check the results later or trigger automated post-processing.

:::

---

## Job Execution Timeline

The following explains exactly what happens when you submit a Tapis app (runtime type ZIP or Singularity/Apptainer) to a [Slurm](https://slurm.schedmd.com/)-based execution system.

At a high level, Tapis does not run jobs itself. It automates what you would otherwise do manually.

```
SSH -> mkdir -> stage files -> write batch script -> sbatch -> poll -> archive
```

Two categories of artifacts define the control boundary.

- Tapis-controlled artifacts include `tapisjob.sh` (scheduler-facing batch wrapper) and `tapisjob.env` (resolved parameters and environment)
- User-controlled artifacts include `tapisjob_app.sh` (your app entrypoint) and everything it loads, runs, installs, or launches

### Step-by-Step

:::{dropdown} 1. From Job Request to Internal Job Record

When you submit a job to a Tapis App

```
POST /v3/jobs
Content-Type: application/json

{
  "appId": "opensees-mp-s3-latest",
  "execSystemId": "stampede3.compute",
  "name": "my-opensees-job",
  "parameterSet": { ... },
  "fileInputs": [ ... ]
}
```

Tapis

1. Validates the request against the App (required inputs present, parameter types correct, values within allowed ranges).
2. Resolves the execution system (*execSystemId*) and checks whether the system's allowed runtimes include the App's *runtime* (ZIP vs SINGULARITY) and whether the system supports the App's *jobType* (BATCH vs FORK).
3. Creates a job record in its database with a Job UUID, App + system references, initial status (*PENDING*), and a copy of the effective ArgString / command line it will eventually run.

At this point, nothing has touched the HPC cluster yet. That happens in the next stage.

:::

:::{dropdown} 2. Establishing SSH Context on the Execution System

Once the job is ready to be launched, the Jobs service

1. Looks up system-level credentials associated with the Execution System. This is typically an SSH keypair managed by Tapis on behalf of the DesignSafe project, with a mapping from Tapis user identity to HPC account (e.g., a TACC username).

2. Opens an SSH session to the login node of the execution system.

   ```bash
   ssh -i /tapis/keys/system-key \
       <hpc-user>@login.stampede3.tacc.utexas.edu
   ```

   In practice, Tapis uses a pool of SSH connections and reuses them where possible, but conceptually it is a normal SSH login as the target HPC user.

3. Sets the working environment with default shell = */bin/bash* (usually), *$HOME* on the HPC system, and any system-level profile scripts (*.bashrc*, *.bash_profile*) that the cluster loads automatically.

From TACC's perspective, this is just another user logging in and running commands.

This happens on the login node for orchestration, not computation.

:::

:::{dropdown} 3. Directory Creation on the Execution System

Tapis creates a structured job workspace on the execution system. The exact paths are defined in the Execution System JSON, but conceptually

```bash
# Job root (execution directory)
JOB_ROOT=/work2/<proj>/<user>/tapis/jobs/${JOB_UUID}

# Tapis often keeps input/output subdirs as well
INPUT_DIR=$JOB_ROOT/input
OUTPUT_DIR=$JOB_ROOT/output
LOG_DIR=$JOB_ROOT

mkdir -p "$JOB_ROOT" "$INPUT_DIR" "$OUTPUT_DIR"
```

These directories correspond to

- *execSystemExecDir* maps to *$JOB_ROOT*
- *execSystemInputDir* maps to *$INPUT_DIR*
- *execSystemOutputDir* maps to *$OUTPUT_DIR*

On TACC systems, all of these live on a shared parallel filesystem (*/work2*, */scratch*, etc.), so every compute node can see them.

This happens on the login node to isolate inputs, runtime, and outputs.

:::

:::{dropdown} 4. Input Staging (SSH + Tapis Files Subsystem)

Tapis copies the user's input files into the job's input directory. There are a few cases.

**4.1. Inputs Already on the Execution System**

If a file input has a path like

```json
"sourceUrl": "tapis://stampede3.compute/work2/XXXXX/username/data/model.tcl"
```

Tapis maps this to a local path on the HPC filesystem and issues an SSH command.

```bash
cp /work2/XXXXX/username/data/model.tcl \
   /work2/XXXXX/username/tapis/jobs/${JOB_UUID}/input/
```

For directories, it uses *cp -r* or *rsync* (implementation detail), but the idea is that the job's input dir collects everything in one place.

**4.2. Inputs Coming from Another Tapis System (e.g., DesignSafe Storage)**

If the file is stored on a different Tapis system or a generic HTTP(S) URL (e.g., S3 URL), the Jobs service hands the work off to the Files/Streams subsystem, which either copies data server-to-server (without shipping it through the client) or streams it from the external source into the *execSystemInputDir* path over SSH/SCP.

Conceptually

```bash
curl -o $INPUT_DIR/model.tcl https://...
# or
scp data-host:/path/to/model.tcl $INPUT_DIR
```

All the inputs end up sitting under *$INPUT_DIR* on a shared filesystem.

A common source of slowdowns is large numbers of small files staged individually. Best practices include zipping/tarring inputs, reusing shared data already in `work` or `scratch`, and keeping long-lived common files outside the job directory.

:::

:::{dropdown} 5. Preparing the Runtime Environment (ZIP vs Singularity/Apptainer)

Tapis applies the App's *runtime* and *containerImage* settings.

**5.1. ZIP Runtime**

For *runtime: "ZIP"*, *containerImage* is something like

```json
"containerImage": "/work2/XXXXX/username/apps/opensees-mp-s3/opensees-mp-s3.zip"
```

Tapis copies and unpacks the archive.

```bash
cp /work2/XXXXX/username/apps/opensees-mp-s3/opensees-mp-s3.zip \
   /work2/XXXXX/username/tapis/jobs/${JOB_UUID}/

cd /work2/XXXXX/username/tapis/jobs/${JOB_UUID}
unzip opensees-mp-s3.zip
```

What is inside the ZIP is entirely up to the App author. It may contain shell scripts, Tcl files, Python scripts, small binaries, templates, etc. There is no container at this point. Everything runs directly in the HPC environment with modules loaded.

**5.2. Singularity / Apptainer Runtime**

For *runtime: "SINGULARITY"*, *containerImage* might be

```json
"containerImage": "/work2/XXXXX/username/containers/opensees-env.sif"
```

Tapis ensures that the *.sif* file is present (copying it if needed). It then plans to use Apptainer with appropriate bind mounts. The actual execution is deferred to the batch script, but Tapis determines the path to the *.sif* file and the directories that must be bound into the container (*$JOB_ROOT*, *$INPUT_DIR*, *$OUTPUT_DIR*).

A typical command line in the Slurm script looks like

```bash
apptainer run \
  --bind $JOB_ROOT:/TapisExec \
  --bind $INPUT_DIR:/TapisInput \
  --bind $OUTPUT_DIR:/TapisOutput \
  /work2/XXXXX/username/containers/opensees-env.sif \
  ./run-opensees-mp.sh "${PARAMS[@]}"
```

Tapis does not run Apptainer. Slurm does, via the batch script. Tapis decides what to mount and which *.sif* to use, then encodes that in the launch script.

:::

:::{dropdown} 6. Generate Launch Artifacts (Wrapper + Environment)

Tapis generates

- `tapisjob.sh`, the scheduler-facing Slurm batch script
- `tapisjob.env`, the resolved parameters, paths, and environment variables

These files are often archived for reproducibility and debugging. You can inspect exactly what Tapis submitted.

:::

:::{dropdown} 7. Building the Slurm Launch Script

At this stage, Tapis has a job directory (*$JOB_ROOT*), inputs staged (*$INPUT_DIR*), and either a ZIP unpacked into *$JOB_ROOT* or a *.sif* image accessible from the cluster.

Now it generates the batch script for *jobType: "BATCH"*.

Tapis writes the batch script line-by-line over SSH.

```bash
cat > /work2/XXXXX/username/tapis/jobs/${JOB_UUID}/tapis_slurm_script.sh << 'EOF'
#!/bin/bash
#SBATCH -J my-opensees-job
#SBATCH -o tapisjob.out
#SBATCH -e tapisjob.err
#SBATCH -N 2
#SBATCH --ntasks-per-node=56
#SBATCH -t 02:00:00
#SBATCH -p normal

set -e
set -o pipefail

JOB_ROOT=/work2/XXXXX/username/tapis/jobs/${JOB_UUID}
INPUT_DIR=$JOB_ROOT/input
OUTPUT_DIR=$JOB_ROOT/output

cd $JOB_ROOT

# Load modules for ZIP runtime
module purge
module load hdf5
module load opensees
module list

echo "Starting job at $(date)" > $JOB_ROOT/tapisjob.log

# ---- Runtime-specific execution ----

# Example A (ZIP runtime)
./run-opensees-mp.sh \
   --input $INPUT_DIR/model.tcl \
   --output $OUTPUT_DIR \
   --np $SLURM_NTASKS \
   >> $JOB_ROOT/tapisjob.log 2>&1

# Example B (Singularity runtime)
# apptainer run --bind ... opensees-env.sif ./run-opensees-mp.sh ...

# -----------------------------------

EXIT_CODE=$?
echo $EXIT_CODE > $JOB_ROOT/tapisjob.exitcode
echo "Job finished with exit code $EXIT_CODE at $(date)" >> $JOB_ROOT/tapisjob.log

EOF

chmod +x /work2/XXXXX/username/tapis/jobs/${JOB_UUID}/tapis_slurm_script.sh
```

A few things to notice.

- Tapis writes the script via a here-document (*cat << 'EOF'*) over SSH
- The script encapsulates scheduler directives (*#SBATCH* lines), module loading, runtime-specific command (ZIP vs Singularity), and a convention for recording exit codes in *tapisjob.exitcode*

This is the same script you would write manually, just generated automatically.

:::

:::{dropdown} 8. Submitting the Job to Slurm (sbatch)

To launch the job, Tapis issues

```bash
cd /work2/XXXXX/username/tapis/jobs/${JOB_UUID}
sbatch tapis_slurm_script.sh
```

Slurm responds with something like

```text
Submitted batch job 9834756
```

Tapis parses the Slurm job ID (*9834756*) and stores it in the Tapis job record. From now on, the authoritative compute-side state lives in the scheduler.

The job is now in the SLURM queue, waiting for the requested resources.

:::

:::{dropdown} 9. Scheduler Execution on Compute Nodes

When scheduled, Slurm runs the batch script on allocated compute nodes.

SSH, staging, and submission happen on the login/service side. Heavy computation happens on compute nodes.

This is where your application actually runs.

:::

:::{dropdown} 10. Handoff from Tapis Wrapper to Your App Wrapper

On compute nodes, `tapisjob.sh` sources `tapisjob.env`, changes into the execution directory, and invokes your app entrypoint (commonly `./tapisjob_app.sh`).

This boundary is crucial.

- Tapis controls `tapisjob.sh` and `tapisjob.env`
- The app author controls `tapisjob_app.sh` and everything it does (modules, MPI launch, Python envs, containers, etc.)

```
tapisjob.sh       <- Tapis-controlled
------------------
tapisjob_app.sh   <- YOU
```

:::

:::{dropdown} 11. Monitoring Job State

While the job is pending or running, Tapis periodically reconnects via SSH and queries Slurm.

```bash
squeue -j 9834756 -h -o "%T"
# or for more detail
sacct -j 9834756 --format=JobIDRaw,State,ExitCode
```

States are mapped into Tapis job states.

- Slurm *PENDING* maps to Tapis *QUEUED*
- Slurm *RUNNING* maps to Tapis *RUNNING*
- Slurm *COMPLETED* maps to Tapis *FINISHED* (assuming exit code 0)
- Slurm *FAILED* / *CANCELLED* maps to Tapis *FAILED* (with details)

Tapis may also inspect *tapisjob.out*, *tapisjob.err*, running Apptainer processes (if applicable), or check for the presence of *tapisjob.exitcode*.

```bash
ls -l tapisjob.out tapisjob.err tapisjob.log
tail -n 40 tapisjob.log
```

:::

:::{dropdown} 12. Completion, Exit Codes, and Error Handling

Once Slurm reports a terminal state (*COMPLETED*, *FAILED*, *CANCELLED*, etc.), Tapis

1. Reads *tapisjob.exitcode* if it exists.

   ```bash
   cat /work2/XXXXX/username/tapis/jobs/${JOB_UUID}/tapisjob.exitcode
   ```

2. If *tapisjob.exitcode* is missing, falls back to the Slurm exit code from *sacct*.

3. Enumerates files in the output directory.

   ```bash
   ls -l /work2/XXXXX/username/tapis/jobs/${JOB_UUID}/output/
   ```

4. Harvests output metadata, collecting file names and sizes, timestamps, and any additional metadata the system can provide.

Tapis then updates the job record. *status = FINISHED* if exit code == 0. *status = FAILED* if exit code != 0 or Slurm reported a failure.

If something goes wrong in earlier stages (input staging, script generation, sbatch failure), Tapis marks the job as *FAILED* with an appropriate reason and error message in its metadata, but the failure is still surfaced via the same job record.

:::

:::{dropdown} 13a. Archiving Outputs (File-Transfer Phase #2)

Tapis transfers outputs to the configured archive system/path.

Common sources of slowdowns include very large output directories and thousands of small files.

Best practices include bundling outputs (tar/zip), filtering aggressively, and writing intermediate data to work/scratch for later collection.

:::

:::{dropdown} 13b. Output Retrieval (User's Perspective)

Once the job is *FINISHED*

- Users can call *GET /v3/jobs/{jobUuid}* to see job metadata, Slurm/job status, and output file listing
- Each output file is available via the Files API, which uses SSH/SCP under the hood to stream the file contents

Conceptually, Files calls do something like

```bash
# For a download
cat /work2/XXXXX/username/tapis/jobs/${JOB_UUID}/output/results.h5
```

wrapped in HTTP responses.

:::

:::{dropdown} 14. Final Metadata + Job Closeout

Tapis records exit code, Slurm status, runtime duration, output file list, and available system logs.

Accessible via

```
GET /v3/jobs/{jobUuid}
```

No further SSH occurs unless files are requested.

:::

:::{dropdown} 15. Optional File Retrieval via Files Service

When a user requests output content

```
GET /v3/files/content?path=/work2/.../job-12345/output/results.h5
```

Tapis either streams directly from the filesystem or uses SCP/SSH internally (system-dependent).

:::

### Condensed Shell View

:::{dropdown} All SSH commands in order

A single Tapis job triggers SSH commands that look roughly like this.

```bash
# 1. Directories
mkdir -p $JOB_ROOT $INPUT_DIR $OUTPUT_DIR

# 2. Input staging
cp /some/existing/path $INPUT_DIR/
# or scp/curl from remote source

# 3. Runtime prep
cp /path/to/app.zip $JOB_ROOT/
cd $JOB_ROOT && unzip app.zip
# or ensure /path/to/image.sif exists

# 4. Slurm script
cat > $JOB_ROOT/tapis_slurm_script.sh << 'EOF'
  #!/bin/bash
  #SBATCH ...
  ...
EOF
chmod +x $JOB_ROOT/tapis_slurm_script.sh

# 5. Submit
cd $JOB_ROOT
sbatch tapis_slurm_script.sh

# 6. Polling
squeue -j <slurmID> -h -o "%T"
# eventually, sacct for final state + exit code

# 7. Completion
ls -l $OUTPUT_DIR
cat tapisjob.exitcode
```

Everything else (the REST API, the App schema, the Job JSON) sits on top of this fairly standard SSH + scheduler automation.

:::

### Condensed Timeline

```
SSH -> mkdir job directories
SSH + Files -> stage inputs
SSH -> unpack ZIP or locate .sif
SSH -> write Slurm script
SSH -> sbatch
Slurm -> execute on compute nodes
SSH -> monitor via squeue
SSH -> collect output metadata
Files Service -> deliver outputs
```

---

## Swim-Lane Diagram

A Tapis job moves vertically in time, while responsibility splits horizontally across four lanes.

```
+------------------+----------------------+---------------------+----------------------+
|   You (User)     |   Tapis Services     |   Login Node        |   Compute Nodes      |
+------------------+----------------------+---------------------+----------------------+
| Submit job       | Validate job         |                     |                      |
| (CLI / API / UI) | Resolve inputs       |                     |                      |
|                  | Open SSH             | SSH session active  |                      |
|                  | Create directories   | mkdir /work/...     |                      |
|                  | Stage inputs         | cp / file transfer  |                      |
|                  | Unpack ZIP / locate  | unzip / locate .sif |                      |
|                  | Write Slurm script   | tapisjob.sh written |                      |
|                  | sbatch               | sbatch invoked      |                      |
|                  | Monitor job          | squeue polling      |                      |
|                  |                      |                     | Job starts           |
|                  |                      |                     | tapisjob.sh runs     |
|                  |                      |                     | -> tapisjob_app.sh   |
|                  |                      |                     | Your computation     |
|                  |                      |                     | runs here            |
|                  | Detect completion    |                     | Job ends             |
|                  | Harvest metadata     | ls output dir       |                      |
|                  | Archive outputs      | file transfer out   |                      |
| Retrieve outputs | Files service        |                     |                      |
+------------------+----------------------+---------------------+----------------------+
```

The login node handles orchestration only (SSH, file staging, script generation, submission). Compute nodes handle all real computation (MPI, OpenSees, Python, containers, etc.). Tapis never executes your science. It automates and observes.

---

## File Transfer Optimization

A significant fraction of total job time may be spent *outside* the RUNNING phase, particularly during file transfers at the beginning and end of a Tapis job. Tapis stages files serially. Thousands of small files are far worse than a few large ones. SSH + metadata operations dominate, not bandwidth.

### Interpreting Job State Durations

Some common patterns you may observe.

- Long time in *PENDING* or *QUEUED* suggests the requested resources may be too large, or the queue may be heavily loaded
- Fast *RUNNING* but slow *ARCHIVING* suggests your workflow is producing too many files or very large outputs
- Delays before the job starts running suggest that input file staging (uploads, copies, or decompression) may be the bottleneck
- Immediate *FAILED* often points to missing input files, incorrect paths, environment setup issues, or permissions problems

### Transfer Best Practices

:::{dropdown} 1. Minimize the number of transferred files

Prefer fewer, larger files and structured directories over flat folders with many files.

Avoid thousands of small text or CSV files, and avoid repeatedly transferring the same static inputs.

:::

:::{dropdown} 2. Keep common inputs in Work or scratch storage

For shared or reusable inputs (e.g., ground-motion sets, large mesh files, lookup tables), store them once in your Work directory or system scratch. Reference them directly from your job script instead of re-uploading.

Scratch and Work spaces may be cleaned periodically. You are responsible for verifying files exist before running, re-populating them when needed, and treating scratch as performance space, not archival storage.

:::

:::{dropdown} 3. Stage results strategically

Instead of archiving everything at the end

- Write intermediate or temporary files to Work or scratch during the run
- Archive only final or essential outputs
- Perform aggregation or post-processing before archiving

This is especially effective now that Work is accessible from Jupyter, enabling you to inspect results interactively, run post-processing notebooks, and collect or compress outputs after the job finishes.

:::

:::{dropdown} 4. Use ZIP files intentionally

Zipped files can dramatically reduce transfer overhead.

For inputs, upload a single ZIP file and unzip inside your job script.

For outputs, package results into one or a few ZIP files and archive those instead of thousands of files.

:::

### I/O Checklist

:::{dropdown} Before you submit a Tapis job, work through this checklist

1. Profile the full lifecycle

   - [ ] Did you look at STAGING_INPUTS and ARCHIVING, not just RUNNING?
   - [ ] Are you tracking totals across multiple jobs to see what is "normal" for your workflow?

2. Reduce file count (this is usually #1)

   - [ ] Can you replace "many small files" with fewer larger files?
   - [ ] Can outputs be aggregated (one CSV/HDF5/JSON per job rather than thousands)?

3. Handle common inputs intelligently

   - [ ] Are shared inputs (e.g., ground motions) stored once in Work/scratch instead of re-staged every run?
   - [ ] Do you have a simple "existence check" step so jobs fail fast if scratch was cleaned?
   - [ ] Do you have a plan to periodically refresh scratch if it is purged?

4. Stage outputs strategically

   - [ ] Are you writing bulky intermediates to Work/scratch during RUNNING?
   - [ ] Are you archiving only final / essential deliverables?
   - [ ] Are you planning to collect/post-process afterwards in Jupyter (since Work is accessible)?

5. Use ZIPs on purpose

   - [ ] Do you ZIP inputs (configs, tables, many small files) and unzip on the compute node?
   - [ ] Do you ZIP outputs (or chunk into multiple ZIPs if very large) before archiving?

6. Sanity checks

   - [ ] Did you request reasonable wall time so the job does not get killed mid-run (wasting staging cost)?
   - [ ] Are logs (stdout/stderr) small and readable (so debugging does not require downloading huge trees)?

:::

---

## Performance Tuning Priorities

Not all stages are equal. The table below lists the highest-impact tuning points, ranked roughly by how often they dominate wall-clock time in real workflows.

| Stage | Risk | What to optimize |
|---|---|---|
| App wrapper (`tapisjob_app.sh`) | Very High | MPI launch, I/O patterns, environment setup, logging verbosity |
| Input staging | High | File count, reuse of shared data, bundling into archives |
| Slurm resource requests | High | Nodes, memory, walltime (over-requesting causes long queue waits) |
| Output archiving | High | Bundle outputs, archive only final artifacts, avoid tiny files |
| Monitoring | Low | Rarely a performance issue |

If a Tapis job is slow, the cause is usually file movement or app-level decisions, not the scheduler itself. Profile first, then scale. One well-profiled job beats 100 blindly submitted ones.

### App Wrapper Tuning

Everything below the handoff line is your responsibility.

```
tapisjob.sh       <- Tapis-controlled
------------------
tapisjob_app.sh   <- YOU
```

Common tuning opportunities include avoiding repeated module loads, pre-installing Python packages (do not pip install every run), using node-local scratch for temporary files, launching MPI correctly (`srun` vs `mpirun`), and controlling logging verbosity. 90% of runtime inefficiencies live in the app wrapper.

---

## Tapis Documentation Links

- [Tapis Project Page](https://tapis-project.org/)
- [Tapis Documentation](https://tapis.readthedocs.io/en/latest/index.html)
- [Tapipy Documentation](https://tapis.readthedocs.io/en/latest/technical/pythondev.html)
- [Tapipy GitHub (OpenAPI specifications)](https://github.com/tapis-project/tapipy/blob/main/tapipy/resources/openapi_v3-apps.yml)
