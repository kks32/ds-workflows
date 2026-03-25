# Job Execution Details

This section explains exactly what happens when you submit a Tapis app (runtime type ZIP or Singularity/Apptainer) to a Slurm-based execution system.

You do not need to know this to *use* Tapis, but understanding it is invaluable for

* debugging failed or slow jobs
* developing custom Tapis apps
* profiling performance and file-transfer overhead
* reasoning about where time is actually spent

At a high level, Tapis does not run jobs itself.
It automates what you would otherwise do manually.

```
SSH -> mkdir -> stage files -> write batch script -> sbatch -> poll -> archive
```

What follows is the full, end-to-end execution timeline, from submission to close-out.

---

## Conceptual Overview (Big Picture)

When you submit a Tapis job

* Service-side orchestration happens on the login node (SSH, directory creation, file staging, script generation, submission)
* Actual computation happens on compute nodes (Slurm executes your batch script, and your application runs here, not on the login node)
* Archiving and file delivery happen after completion

Two key artifacts define the control boundary.

* Tapis-controlled artifacts include `tapisjob.sh` (scheduler-facing batch wrapper) and `tapisjob.env` (resolved parameters and environment)
* User-controlled artifacts include `tapisjob_app.sh` (your app entrypoint) and everything it loads, runs, installs, or launches

---

## Detailed Execution Timeline

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

   In practice, Tapis uses a pool of SSH connections and reuses them where possible, but conceptually it's a normal SSH login as the target HPC user.

3. Sets the working environment with default shell = */bin/bash* (usually), *$HOME* on the HPC system, and any system-level profile scripts (*.bashrc*, *.bash_profile*) that the cluster loads automatically.

From TACC's perspective, this is just another user logging in and running commands.

This happens on the login node for orchestration, not computation.

:::

:::{dropdown} 3. Directory Creation on the Execution System

Next, Tapis creates a structured job workspace on the execution system. The exact paths are defined in the Execution System JSON, but conceptually

```bash
# Job root (execution directory)
JOB_ROOT=/work2/<proj>/<user>/tapis/jobs/${JOB_UUID}

# Tapis often keeps input/output subdirs as well:
INPUT_DIR=$JOB_ROOT/input
OUTPUT_DIR=$JOB_ROOT/output
LOG_DIR=$JOB_ROOT

mkdir -p "$JOB_ROOT" "$INPUT_DIR" "$OUTPUT_DIR"
```

These directories correspond to

* *execSystemExecDir* maps to *$JOB_ROOT*
* *execSystemInputDir* maps to *$INPUT_DIR*
* *execSystemOutputDir* maps to *$OUTPUT_DIR*

On TACC systems, all of these are on a shared parallel filesystem (*/work2*, */scratch*, etc.), so every compute node can see them.

This happens on the login node to isolate inputs, runtime, and outputs.
:::

:::{dropdown} 4. Input Staging (SSH + Tapis Files subsystem)

Tapis now needs to copy the user's input files into the job's input directory. There are a few cases.

**4.1. Inputs Already on the Execution System**

If a file input has a path like

```json
"sourceUrl": "tapis://stampede3.compute/work2/XXXXX/username/data/model.tcl"
```

Tapis maps this to a local path on the HPC filesystem and issues an SSH command.

``` bash
cp /path/to/source /work2/.../job-12345/input/
```
for example
```bash
cp /work2/XXXXX/username/data/model.tcl \
   /work2/XXXXX/username/tapis/jobs/${JOB_UUID}/input/
```

For directories, it uses *cp -r* or *rsync* (implementation detail), but the idea is that the job's input dir collects everything in one place.

**4.2. Inputs Coming from Another Tapis System (e.g., DesignSafe Storage)**

If the file is stored on a different Tapis system (e.g., a DesignSafe data system) or a generic HTTP(S) URL (e.g. S3 URL), the Jobs service hands the work off to the Files/Streams subsystem, which either copies data server-to-server (without shipping it through the client), or streams it from the external source into the *execSystemInputDir* path over SSH/SCP.

Conceptually, you can think of it as

```bash
curl -o $INPUT_DIR/model.tcl https://...
# or
scp data-host:/path/to/model.tcl $INPUT_DIR
```

The actual mechanism is more integrated, but the result is straightforward. All the inputs end up sitting under *$INPUT_DIR* on a shared filesystem.

A common source of slowdowns is large numbers of small files staged individually.

Best practices include zipping/tarring inputs, reusing shared data already in `work` or `scratch`, and keeping long-lived common files outside the job directory.

:::

:::{dropdown} 5. Preparing the Runtime Environment (ZIP vs Singularity/Apptainer)

Now Tapis applies the App's *runtime* and *containerImage* settings.

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
# or tar -xzf app.tgz, depending on the archive type
```

What's inside the ZIP is entirely up to the App author. It may contain shell scripts, Tcl files, Python scripts, small binaries, templates, etc.

* There is no container at this point.
* Everything will run directly in the HPC environment (with modules loaded).
* Modules are loaded in the batch script or app wrapper.

---

**5.2. Singularity / Apptainer Runtime**

For *runtime: "SINGULARITY"*, *containerImage* might be

```json
"containerImage": "/work2/XXXXX/username/containers/opensees-env.sif"
```

Tapis ensures that the *.sif* file is present (copying it if needed). It then plans to use Apptainer with appropriate bind mounts. The actual execution is deferred to the batch script, but Tapis determines the path to the *.sif* file and the directories that must be bound into the container (execution directory *$JOB_ROOT*, input directory *$INPUT_DIR*, output directory *$OUTPUT_DIR*).

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

:::{dropdown} 6. Generate Launch Artifacts (wrapper + environment)

Tapis generates

* `tapisjob.sh`, the scheduler-facing Slurm batch script
* `tapisjob.env`, the resolved parameters, paths, and environment variables

These files are often archived for reproducibility and debugging.

You can inspect exactly what Tapis submitted.

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

# Load modules for ZIP runtime, or for inside-container tooling if needed
module purge
module load hdf5
module load opensees
module list

echo "Starting job at $(date)" > $JOB_ROOT/tapisjob.log

# ---- Runtime-specific execution ----

# Example A: ZIP runtime
# (The ZIP unpacked a wrapper script run-opensees-mp.sh)
./run-opensees-mp.sh \
   --input $INPUT_DIR/model.tcl \
   --output $OUTPUT_DIR \
   --np $SLURM_NTASKS \
   >> $JOB_ROOT/tapisjob.log 2>&1

# Example B: Singularity runtime
# apptainer run --bind ... opensees-env.sif ./run-opensees-mp.sh ...

# -----------------------------------

EXIT_CODE=$?
echo $EXIT_CODE > $JOB_ROOT/tapisjob.exitcode
echo "Job finished with exit code $EXIT_CODE at $(date)" >> $JOB_ROOT/tapisjob.log

EOF

chmod +x /work2/XXXXX/username/tapis/jobs/${JOB_UUID}/tapis_slurm_script.sh
```

A few things to notice.

* Tapis writes the script via a here-document (*cat << 'EOF'*) over SSH.
* The script itself encapsulates scheduler directives (*#SBATCH* lines), module loading, runtime-specific command (ZIP vs Singularity), and a convention for recording exit codes in *tapisjob.exitcode*.

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

* Tapis controls `tapisjob.sh` and `tapisjob.env`
* The app author controls `tapisjob_app.sh` and everything it does (modules, MPI launch, Python envs, containers, etc.)

:::


:::{dropdown} 11. Monitoring Job State

While the job is pending or running, Tapis periodically reconnects via SSH and queries Slurm.

```bash
squeue -j 9834756 -h -o "%T"
# or for more detail:
sacct -j 9834756 --format=JobIDRaw,State,ExitCode
```

States are mapped into Tapis job states (QUEUED, RUNNING, FINISHED, FAILED).

* Slurm *PENDING* maps to Tapis *QUEUED*
* Slurm *RUNNING* maps to Tapis *RUNNING*
* Slurm *COMPLETED* maps to Tapis *FINISHED* (assuming exit code 0)
* Slurm *FAILED* / *CANCELLED* maps to Tapis *FAILED* (with details)


It may also inspect *tapisjob.out*, *tapisjob.err*, running Apptainer processes (if applicable), or check for the presence of *tapisjob.exitcode*.

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

2. If *tapisjob.exitcode* is missing, it falls back to the Slurm exit code from *sacct*.

3. It enumerates files in the output directory.

   ```bash
   ls -l /work2/XXXXX/username/tapis/jobs/${JOB_UUID}/output/
   ```

4. Harvests output metadata. It collects file names and sizes, timestamps, and any additional metadata the system can provide.

Tapis then updates the job record. *status = FINISHED* if exit code == 0. *status = FAILED* if exit code != 0 or Slurm reported a failure.

If something goes wrong in earlier stages (input staging, script generation, sbatch failure), Tapis marks the job as *FAILED* with an appropriate reason and error message in its metadata, but the failure is still surfaced via the same job record.

:::

:::{dropdown} 13a. Archiving Outputs (file-transfer phase #2)

Tapis transfers outputs to the configured archive system/path.

Common sources of slowdowns include very large output directories and thousands of small files.

Best practices include bundling outputs (tar/zip), filtering aggressively, and writing intermediate data to work/scratch for later collection.

:::

:::{dropdown} 13b. Output Retrieval (User's Perspective)

From the user side, once the job is *FINISHED*

* Users can call *GET /v3/jobs/{jobUuid}* to see job metadata, Slurm/job status, and output file listing.
* Each output file is available via the Files API, which again uses SSH/SCP under the hood to stream the file contents.

Conceptually, Files calls do something like

```bash
# For a download:
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

Tapis either streams directly from the filesystem, or uses SCP/SSH internally (system-dependent).

:::

---
:::{dropdown} Condensed Shell View

A single Tapis job triggers SSH commands that look roughly like this (in order).

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

---

## Condensed Timeline (Reference)

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


## Visual Swim-Lane Diagram

Think of a Tapis job as moving vertically in time, while responsibility is split horizontally across four lanes.

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

### Clarification Reinforced

* The login node handles orchestration only (SSH, file staging, script generation, submission)
* Compute nodes handle all real computation (MPI, OpenSees, Python, containers, etc.)
* Tapis never executes your science. It automates and observes.

---

## Where Performance Tuning Matters Most

Not all stages are equal.
Below are the highest-impact tuning points, ranked roughly by how often they dominate wall-clock time in real workflows.

---

### Input Staging (Often the Hidden Bottleneck)

Tapis stages files serially. Thousands of small files are far worse than a few large ones. SSH + metadata operations dominate, not bandwidth.

Symptoms include jobs appearing "stuck" in `PENDING` or `STAGING`, and very short compute runs but long total job time.

Best practices include bundling inputs (`tar.gz`, `zip`), keeping shared datasets in `work` or `scratch`, avoiding repeatedly staging identical ground motions, and staging once to reuse many times.

---

### Slurm Script Generation and Resource Mapping

Incorrect resource requests waste queue time. Over-requesting nodes means long queue waits. Under-requesting memory means runtime failure.

What to tune includes nodes vs tasks vs threads, memory per node, and walltime realism.

> Profile first, then scale. One well-profiled job beats 100 blindly submitted ones.

---

### Your App Wrapper (`tapisjob_app.sh`)

This is the most important boundary.

Everything below this line is your responsibility.

```
tapisjob.sh  <- Tapis-controlled
----------------
tapisjob_app.sh  <- YOU
```

Common tuning opportunities include avoiding repeated module loads, pre-installing Python packages (don't pip install every run), using node-local scratch for temporary files, launching MPI correctly (`srun` vs `mpirun`), and controlling logging verbosity.

> 90% of runtime inefficiencies live here.

---

### Output Archiving (File-Transfer Phase #2)

Archiving happens after your job finishes. Users often forget this counts toward perceived job time. Thousands of tiny files are disastrous.

Best practices include writing raw outputs to scratch/work, collecting and bundling at the end, archiving only final artifacts, and avoiding archiving intermediate checkpoints unless needed.

---

### Monitoring Overhead (Usually Minor)

Polling via `squeue` is lightweight. Rarely a performance issue unless job states flap rapidly.

---

## Performance Tuning Cheat Sheet

| Stage | Risk | What to optimize |
| ---------------- | ------------ | --------------------------- |
| Input staging | High | File count, reuse, bundling |
| Slurm requests | High | Nodes, memory, walltime |
| App wrapper | Very High | MPI launch, I/O, env setup |
| Output archiving | High | Bundle outputs |
| Monitoring | Low | Rarely critical |

---

## One-Sentence Mental Model

> If a Tapis job is slow, it's usually not Slurm. It's file movement or app-level decisions.
