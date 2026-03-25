# Execution Strategies
*How workloads are mapped onto compute systems*

An execution strategy describes *how* a workload is launched, distributed, coordinated, and completed on a computing system. While the workload defines computational behavior, the execution strategy defines control flow, parallel structure, and resource usage.

Execution strategies are independent of tools. The same strategy may be implemented using JupyterHub, SLURM scripts, or Tapis apps. What changes is *automation*, not intent.


---

## The Role of Execution Strategies

Execution strategies sit between scientific intent and computing tools.

They answer questions such as

* Should tasks run independently or in coordination?
* Should the workflow be one long job or many small jobs?
* Is performance limited by CPU, memory, communication, or I/O?
* Does scaling mean more tasks, larger tasks, or longer runs?

Understanding execution strategies prevents common pitfalls such as

* oversubscribing memory,
* underutilizing nodes,
* overwhelming the filesystem with small files,
* or adding resources that actually *reduce* performance.


---

## The Three Core Execution Dimensions

Every execution strategy is shaped by how a workload behaves along three fundamental dimensions.

:::{dropdown} 1. Task Independence

Do individual tasks depend on each other?

* Independent tasks can run in any order or in parallel
  *(Monte Carlo, parameter sweeps)*
* Dependent tasks must run in sequence or coordinated steps
  *(time-marching simulations, multi-stage workflows)*

:::

:::{dropdown} 2. Resource Coupling

Do tasks share memory or communicate frequently?

* Loosely coupled tasks involve minimal communication and file-based exchange
* Tightly coupled tasks require frequent synchronization, shared state, and MPI

:::

:::{dropdown} 3. Time Structure

How does the workload evolve over time?

* Short-lived tasks produce many fast jobs where scheduling overhead matters
* Long-running tasks require stability, checkpointing, and walltime management
* Iterative tasks involve repeated execution with evolving state

:::

These dimensions, not the software, determine the correct execution strategy.

A single workflow may change execution strategy over its lifetime. For example, it might start as embarrassingly parallel during exploration and evolve into tightly coupled execution at scale.

---

### Common Execution Strategies

Below are the most common execution strategies used on DesignSafe and similar HPC platforms.

:::{dropdown} 1. Embarrassingly Parallel Execution

Best for Monte Carlo, parameter sweeps, and batch preprocessing.

* Each task runs independently
* Minimal memory per task
* Scales horizontally across many cores or nodes

Typical patterns include

* Job arrays
* Parameterized batch jobs
* Task launchers

Primary risk is scheduling overhead and file I/O explosion.

:::

:::{dropdown} 2. Single Large Batch Execution

Best for stepwise simulations and long-running solvers.

* One job runs for a long time
* Memory footprint is stable
* Parallelism is moderate or internal

Typical patterns include

* Single SLURM job
* Multi-core shared-memory execution
* Checkpoint/restart cycles

Primary risk is underutilization if parallelism is limited.

:::

:::{dropdown} 3. Tightly Coupled MPI Execution

Best for large structural models and domain-decomposed simulations.

* Tasks exchange data frequently
* Strong synchronization requirements
* Memory and network performance dominate

Typical patterns include

* MPI ranks per node
* Domain decomposition
* Collective communication

Primary risk is communication overhead and load imbalance.

:::

:::{dropdown} 4. Pipeline / Multi-Stage Execution

Best for preprocess, simulate, and postprocess workflows.

* Workload is decomposed into stages
* Each stage may use a different execution strategy
* Intermediate data must be staged carefully

Typical patterns include

* Sequential job chaining
* Workflow managers
* Conditional execution

Primary risk is that data movement dominates runtime.

:::

:::{dropdown} 5. Accelerated Execution (GPU / Specialized Hardware)

Best for ML training, large matrix operations, and some preprocessing.

* High compute intensity
* Performance sensitive to memory layout and data transfer
* Often paired with CPU preprocessing

Typical patterns include

* GPU-enabled batch jobs
* Hybrid CPU/GPU pipelines

Primary risk is idle accelerators due to poor data staging.

:::

---

## Execution Strategy Is Not Platform

An important distinction to keep in mind.

> Execution strategies describe structure, not tools.

The same strategy can be implemented using

* JupyterHub (interactive, exploratory)
* SLURM batch scripts (manual control)
* Tapis apps (automated, repeatable workflows)

The *strategy* stays the same. Only the level of automation and orchestration changes.

---

## Execution Strategy Is Not Resource Size

A common misconception is that scaling a workload means adding more resources.

> Many workloads fail to scale because the *execution strategy* does not match the *workload structure*.

Examples of this mismatch in practice.

* Adding nodes to a tightly coupled simulation may slow it down
* Running many tiny tasks as one job may waste cores
* GPU jobs without sufficient preprocessing may idle accelerators

Choosing the right execution strategy is often more important than choosing the largest system.


---

## Looking Ahead

In later chapters, these execution strategies will be mapped to

* Interactive environments (e.g., JupyterHub)
* Batch systems (SLURM)
* Automated pipelines (Tapis applications)

The goal is not to lock you into a single approach, but to give you a strategy-first mindset for building scalable, reusable computational workflows.


---

## Guiding Principle

> Performance problems are usually strategy problems, not hardware problems.

# Execution Strategy Matrix
*Matching workload behavior to execution structure*

The table below summarizes structural differences between execution strategies, not tool-specific implementations.


| Execution Strategy          | Best-Fit Workloads                                  | Task Coupling | Resource Focus         | Typical Scaling Pattern   | Key Risks                          |
| --------------------------- | --------------------------------------------------- | ------------- | ---------------------- | ------------------------- | ---------------------------------- |
| Embarrassingly Parallel | Monte Carlo, parametric sweeps, batch preprocessing | Independent   | CPU, I/O               | Many small jobs           | Scheduler overhead, file explosion |
| Single Long Batch       | Stepwise simulations, nonlinear solvers             | Sequential    | CPU, memory            | Longer walltime           | Idle cores if poorly parallelized  |
| Tightly Coupled MPI     | Large FE models, domain-decomposed solvers          | Strong        | Memory, network        | Fewer larger jobs         | Communication and load imbalance   |
| Pipeline / Multi-Stage  | Pre, simulate, and post workflows                   | Mixed         | I/O, orchestration     | Stage-by-stage            | Data movement dominates runtime    |
| Accelerated (GPU)       | ML training, dense linear algebra                   | Internal      | GPU memory and bandwidth | Fewer high-intensity jobs | Idle accelerators, poor staging    |

Scaling is not one-dimensional. A strategy that scales well in *number* may scale poorly in *size* or *communication*.

---

## Execution Strategy Decision Tree


Step 1. Are tasks independent?

* Yes, then use embarrassingly parallel execution
* No, then go to Step 2

Step 2. Do tasks communicate frequently?

* Yes, then use tightly coupled (MPI-style) execution
* No, then go to Step 3

Step 3. Is the workload long-running or staged?

* Single long run, then use single large batch
* Multiple stages, then use pipeline / multi-stage execution

Step 4. Is compute dominated by matrix ops or learning loops?

* Yes, then use accelerated (GPU-based) execution
* No, then use CPU-based batch execution

This decision tree is intentionally tool-agnostic. The same strategy can later be implemented using different systems.

---

## Common Strategy Mistakes (and How to Avoid Them)

* Running independent tasks as one big job wastes cores and walltime.

* Adding nodes to tightly coupled jobs without decomposition increases communication overhead.

* Using GPUs without data staging discipline leaves accelerators sitting idle.

* Over-fragmenting pipelines causes file transfer to dominate runtime.

Execution strategy is about matching structure to behavior, not maximizing resources.
