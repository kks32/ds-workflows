# SLURM and Tapis

At this point, you already understand how SLURM works directly. You request resources, submit a batch script, wait in a queue, run on allocated nodes, and collect outputs. What changes when you use Tapis is *not* how jobs run. It is *how workflows are described, launched, managed, and reused*.

Tapis does not replace SLURM.
It orchestrates SLURM.

Think of Tapis as a translation and management layer that converts higher-level job descriptions into the same SLURM mechanics you already know, then tracks, stages, archives, and exposes everything programmatically.

---

## The Conceptual Shift from Scripts to Descriptions

When working directly with SLURM, the batch script is both the definition of the workflow and the mechanism for running it. With Tapis, those responsibilities are deliberately separated.

| If you were using SLURM directly | With Tapis |
| --------------------------------- | ------------------------------------- |
| Write and edit an `sbatch` script | Define a Tapis App once |
| Modify scripts per run | Submit Tapis Jobs with parameters |
| Manually stage inputs | Tapis stages inputs automatically |
| Monitor with `squeue` / `sacct` | Query job state via API, CLI, Python |
| Track outputs yourself | Tapis archives outputs + metadata |

You still request the same resources (nodes, cores, walltime, queues) but you express them declaratively, rather than embedding them repeatedly in scripts.

A helpful way to think about it:

> A Tapis App captures what used to live in a SLURM script.
> A Tapis Job captures what used to change from run to run.

---

## SLURM Concepts vs. Tapis Concepts

This mapping is the key to demystifying Tapis. Nothing here is new. It is simply reorganized.

| SLURM concept | What it controls | Tapis equivalent |
| ------------------------- | ---------------------------- | -------------------------- |
| Batch script (`sbatch`) | Execution logic + directives | Tapis App |
| Job submission (`sbatch`) | Send job to scheduler | Tapis Job submission |
| Partition / queue | Priority & limits | `execSystemLogicalQueue` |
| Nodes | Physical machines | `nodeCount` |
| Tasks / cores | Parallelism | `coresPerNode`, MPI config |
| Walltime | Max runtime | `maxMinutes` |
| Working directory | Runtime location | Tapis execution directory |
| Stdout / stderr | Logs | Archived job logs |

Once you see this correspondence, Tapis stops feeling "magical" and starts feeling like a well-organized automation of SLURM.

---

## What SLURM Controls (and Always Will)

SLURM remains the scheduler of record. It is responsible for

* allocating nodes, cores, and memory
* queuing jobs and enforcing priorities
* launching MPI, serial, and hybrid workloads
* tracking job states (*PENDING*, *RUNNING*, *COMPLETED*, *FAILED*)

If you write SLURM scripts directly, you control everything: resource flags, execution commands, file paths, scratch usage, arrays, and dependencies. This offers maximum flexibility, but also maximum responsibility.

For many users, direct SLURM scripts are the prototype. They are how you understand performance, scaling, and failure modes before automating anything.

---

## What Tapis Adds on Top of SLURM

Tapis sits above the scheduler and automates the surrounding workflow.

* input staging (MyData, Projects, Work, external systems)
* output collection and archiving
* job submission and monitoring
* consistent execution environments
* programmatic access (API, Python, CLI, Web Portal)

When you submit a Tapis job, SLURM still runs it on systems at the Texas Advanced Computing Center (for example on Stampede3), but you no longer have to manage all of the glue yourself.

Tapis automates the mechanics without hiding the truth.

* jobs still wait in queues
* resource limits still apply
* walltime is still enforced
* MPI and threading behave exactly the same

The difference is where intent is expressed.

---

## When Direct SLURM Makes Sense

Using SLURM directly is often the right choice when

* developing or debugging low-level performance behavior
* experimenting interactively on the cluster
* using advanced scheduler features (complex arrays, dependencies)
* building short-lived or highly customized workflows

In these cases, working close to the scheduler is an advantage.

---

## When Tapis Automation Makes Sense

Tapis becomes essential when workflows need to be

* repeatable, for the same run with different inputs
* scalable, across many jobs, many users, many datasets
* shareable, so others can run without reading your scripts
* integrated, so the Web Portal, Jupyter, Python, and CLI all behave consistently

This is especially important for

* ensemble studies
* uncertainty quantification
* parameter sweeps
* production runs
* teaching and training materials

---

## A Healthy Mental Model

A concise way to hold all of this together:

> SLURM defines how jobs run.
> Tapis defines how workflows are launched, managed, and reused.

You don't replace SLURM by learning Tapis.
You encapsulate SLURM inside a more structured workflow system.

---

## The Developer Bridge

For app authors and advanced users, Tapis is best understood as a programmatic SLURM job generator.

* Tapis generates scheduler-facing scripts (e.g., `tapisjob.sh`)
* It exports parameters and paths via environment files (e.g., `tapisjob.env`)
* It submits the job with `sbatch`
* It monitors scheduler state and maps it into Tapis job states
* It invokes your app wrapper (commonly `tapisjob_app.sh`) on the compute nodes
* It archives outputs and preserves logs and metadata

All of this happens using the same SLURM mechanisms you would use manually, just assembled, executed, and tracked automatically.

The detailed timing, file-transfer stages, login-node vs compute-node behavior, and performance implications are covered in the in-depth Tapis execution sections that follow.

---

## A Common (and Recommended) Progression

Most advanced users naturally evolve along this path.

1. Run jobs interactively (Web Portal, JupyterHub)
2. Write SLURM scripts to understand scaling and performance
3. Wrap those scripts into Tapis Apps
4. Automate execution using Python or the CLI
5. Reuse and extend workflows across projects and collaborators

Each step builds on the previous one. Nothing is discarded.

---

## Long-Term Benefits

Research workflows change. Models grow, datasets expand, collaborators rotate, and questions evolve.

Separating execution mechanics (SLURM) from workflow logic (Tapis) is what allows your work to scale without becoming brittle.

That separation is what turns one-off jobs into sustainable computational infrastructure.

# Tapis as Automation

This section explains what Tapis adds on top of SLURM.

Tapis does not change

* how SLURM allocates resources,
* where jobs run,
* or how execution behaves at runtime.

Instead, Tapis automates the login-node orchestration required to submit standard SLURM batch jobs.

---

## What Tapis Is (and Is Not)

* Tapis is a job-submission, staging, and monitoring layer
* Tapis is not a scheduler, executor, or alternative runtime

From SLURM's perspective, a Tapis job is indistinguishable from a manually submitted batch job.

---

:::{dropdown} How Tapis Describes SLURM Resources

When submitting a job through Tapis, resource requirements are specified in either

* the app definition, or
* the job submission request (overrides)

Typical Tapis fields include

* `nodeCount` for the number of compute nodes
* `coresPerNode` for CPU cores per node
* `memoryMB` for memory per node
* `maxMinutes` for the walltime limit
* queue name (e.g., `normal`, `development`)

These fields describe exactly the same resources you would request in a manual SLURM job.

Tapis does not invent new resource concepts. It mirrors SLURM's model.

:::

---

:::{dropdown} How Tapis Becomes a SLURM Job

Tapis converts job attributes into a standard SLURM batch script (`tapisjob.sh`) containing directives such as

```bash
#SBATCH -N <nodeCount>
#SBATCH -n <total_tasks>
#SBATCH -t <walltime>
#SBATCH --mem=<memory>
```

By the time `sbatch` is invoked, SLURM sees the job exactly as if you had written the batch script yourself.

There is no special execution mode, wrapper scheduler, or bypass mechanism.

:::

---

:::{dropdown} How Tapis Submits the Job

For a Tapis job submitted to a SLURM-backed system:

1. Tapis opens an SSH session to a login node
2. Creates the job execution directory
3. Stages input files
4. Writes the SLURM batch script (`tapisjob.sh`)
5. Runs `sbatch tapisjob.sh` on the login node

All of these actions are lightweight orchestration tasks, exactly what login nodes are designed for.

This is the same sequence a user would perform manually.

:::

---

:::{dropdown} What Runs on Compute Nodes

Once `sbatch` is called

* SLURM queues the job
* compute nodes are allocated
* `tapisjob.sh` starts running on a compute node
* it invokes `tapisjob_app.sh`
* your application executes on compute nodes

`tapisjob_app.sh` is responsible for

* loading modules
* activating environments
* setting paths and variables
* launching serial or parallel workloads (MPI, launchers, etc.)

From this point forward, Tapis is no longer involved in execution.

:::

---

:::{dropdown} Environment Setup in Tapis Jobs

Although users interact with DesignSafe via the web portal, JupyterHub, or APIs, none of those environments execute the job.

Compute nodes start clean, so all environment setup must occur inside `tapisjob_app.sh`, for example

```bash
module load python/3.12.11
module load opensees
module load hdf5/1.14.4
```

If something is not defined in the job script, it does not exist at runtime.

This is intentional and critical for

* reproducibility
* isolation
* predictable performance

:::

---

## Practical Implications

* Tapis does not alter SLURM semantics
* Tapis generates and submits standard SLURM batch jobs
* Resource usage and execution follow exactly the same rules as manual SLURM jobs
* Tapis automates login-node workflows, nothing more, nothing less

Once SLURM fundamentals are understood, Tapis becomes a transparent automation layer rather than a black box.
