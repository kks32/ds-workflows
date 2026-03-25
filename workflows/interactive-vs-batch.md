# Workflow Types

Modern computational research spans a wide range of workflows, from exploratory scripting and visualization to large-scale, long-running simulations. DesignSafe provides multiple compute environments to support this spectrum, each optimized for a different style of interaction and scale of computation.

At a high level, these environments fall into two workflow categories.

:::{dropdown} A. Interactive Workflows, designed for *human-in-the-loop* computing
Interactive workflows are designed for *human-in-the-loop* computing. You write code, run it, inspect results, modify inputs, and iterate, often many times in a short period.

Typical characteristics include

* Immediate or near-immediate execution
* Direct access to terminals, editors, and notebooks
* Tight feedback loops
* Ideal for development, debugging, testing, and visualization

Examples include

* Developing Python or Tcl scripts
* Exploring datasets interactively
* Running small to medium test cases
* Automating analyses with immediate feedback

On DesignSafe, interactive workflows are primarily supported through Jupyter-based environments, where computation happens inside containerized sessions.
:::

:::{dropdown} B. Batch Workflows, optimized for *throughput and scale*

Batch workflows are optimized for *throughput and scale*. You prepare a job in advance, submit it to a scheduler, and wait for it to run when resources become available.

Typical characteristics include

* Non-interactive execution
* Explicit resource requests (cores, memory, time)
* Jobs may queue before running
* Designed for large, long-running, or parallel computations

Examples include

* Large simulations
* Parameter sweeps
* MPI-based parallel jobs
* Production or repeated analyses

On DesignSafe and TACC systems, batch workflows are managed by schedulers (such as SLURM) and accessed through Tapis-based job submission, whether via command line, web portal, or automation tools.

:::


Interactive and batch workflows are not mutually exclusive.
Effective computational research often uses both, selecting the environment that best matches the task at hand.

**A Clarification**

Even when launched through a graphical web interface, jobs submitted through the DesignSafe Web Portal are batch jobs. The interface simplifies submission, but the execution model remains queued, scheduled, and non-interactive.

---
### How These Fit Together

Rather than competing options, these workflows are complementary.

* Interactive workflows accelerate learning and development
* Batch workflows enable scale and performance
* Many workflows naturally transition from one to another as projects mature

---
## A Typical Compute Path

Many users naturally follow this progression.

1. Develop and test interactively
2. Validate on small problems
3. Scale up using batch jobs
4. Automate or repeat studies at scale

This transition is expected and supported within the DesignSafe ecosystem.

---

:::{tip}
Interactive and batch workflows are not mutually exclusive.
Effective computational research often uses both, selecting the environment that best matches the task at hand.
:::


---

## Choosing the Right Environment

Understanding *where* your computation runs and *how* it is executed helps you

* Avoid unnecessary queues
* Use resources efficiently
* Scale smoothly from prototypes to production
* Reduce friction as projects grow
