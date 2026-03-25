# Workflow Design
*Designing Computational Workflows Around Your Research Needs*

Your computational workflow should be designed around your research needs, not around a specific tool, platform, or computing system. Different research questions impose very different computational requirements, and those requirements naturally lead to different workflow designs. A workflow that works well for exploratory analysis may fail completely when scaled to thousands of simulations, or when extended to include coupled physics, uncertainty quantification, or machine learning.

The first step in designing an effective workflow is to identify what you are trying to compute, not where you intend to run it.

### A Practical Way to Think About Workflow Design

Rather than building a single, rigid pipeline, it is far more effective to break your workflow into reusable modules, each responsible for a well-defined task. Common modules include

* Input generation (models, parameters, ground motions, meshes)
* Execution (single simulations, ensembles, training loops)
* Post-processing (response extraction, aggregation, visualization)
* Iteration or automation (parameter sweeps, Monte Carlo loops, retraining)

Once your workflow is modular, you can

* Reuse the same components across different projects
* Swap execution environments as your computational demands grow
* Combine modules in new ways as your research questions evolve

This modular approach is what allows workflows to remain adaptable over time, instead of being rewritten from scratch whenever requirements change.

### Designing for Scalability From the Start

Scalability is not something you add later. It must be considered during workflow design.

A workflow that runs successfully for

* one model
* one parameter set
* one dataset

should be able to scale to

* hundreds or thousands of simulations
* larger or higher-resolution models
* more complex coupling or uncertainty

However, scaling is not just about adding more nodes. Different analysis types scale in different ways.

* Some workloads scale trivially by running many independent jobs.
* Others are limited by memory per node, communication costs, or solver structure.
* Some benefit from GPUs; others do not.

DesignSafe supports this diversity by providing multiple execution environments, but it is your workflow design that determines whether those resources can be used effectively.

On DesignSafe, HPC resources are not just for scaling "up" by adding more nodes. They also allow you to adapt to the computational profile of your analysis. Each workload has its own best-fit execution environment.

# Workloads
*Understanding what you run, how it scales, and where bottlenecks emerge*


## Why Workloads Matter

A *workload* is more than a program or script. It is the combined behavior of computation, memory, data movement, and parallelism over time.

Two workflows may run the same software and yet behave very differently on an HPC system depending on *how* they scale, *what* they stress, and *where* bottlenecks occur.

Understanding your workload allows you to

* choose the right execution environment,
* design scalable workflows,
* and avoid performance surprises as your research grows.



To support intentional workflow design, we can separate computational work into two complementary views.


:::{dropdown} 1. Analysis Types describe scientific intent
*What kind of scientific or engineering task you are performing*

The following table provides an overview of the kinds of analyses that users may run on DesignSafe.

| Analysis Type                 | Description                                                                                                                                                      |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Parallel Monte Carlo      | Monte Carlo methods involve running repeated random samples to solve deterministic problems that are too complex for direct computation.                         |
| Parametric Sweeps         | Run a model or simulation across a range of input parameters to study system behavior or performance variation.                                                  |
| ML Training Loops         | ML training involves iterative updates to a model's parameters using a training dataset.                                                                         |
| Coupled Simulations       | Simultaneously simulate multiple interacting physical domains or solvers (e.g., fluid + structure, heat + stress).                                               |
| Stepwise Simulation       | A sequential, time-marching simulation solving physical phenomena across a time domain, e.g., using finite difference, finite element, or finite volume methods. |
| Batch Pre/Post-Processing | Transform or clean a large batch of data files or inputs to prepare for further analysis or training.                                                            |

:::

:::{dropdown} 2. Computation Types describe computational behavior
*How that task stresses memory, CPUs/GPUs, parallelism, and speed*

The table below shows how these analyses differ in their computational characteristics. This helps users match their workload to the right DesignSafe resource (e.g., JupyterHub, HPC batch jobs, or Tapis apps).

| Analysis Type                 | Memory Usage     | CPU vs GPU             | Parallelism Needs | Speed Sensitivity | Reason / Notes                                                                                                  |
| ----------------------------- | ---------------- | ---------------------- | ----------------- | ----------------- | --------------------------------------------------------------------------------------------------------------- |
| Parallel Monte Carlo      | 🔹 Low–Moderate  | ✅ CPU preferred        | 🔹 High           | 🔸 Low–Moderate   | Trivially parallel: each run is independent; minimal memory overhead, great for CPU clusters or grid computing. |
| Parametric Sweeps         | 🔹 Low–Moderate  | ✅ CPU preferred        | 🔹 High           | 🔸 Low–Moderate   | Like Monte Carlo, easily parallelized with little inter-process communication.                                  |
| ML Training Loops         | 🔸 Moderate–High | 🔴 GPU accelerated     | 🔹 Moderate–High  | 🔴 High           | GPU-accelerated for fast matrix ops; training requires high compute, memory usage depends on model size.        |
| Coupled Simulations       | 🔴 High          | ✅ CPU dominated        | 🔸 Low–Moderate   | 🔴 High           | Memory-intensive due to mesh/data exchange between solvers; limited parallelization unless domain-decomposed.   |
| Stepwise Simulation       | 🔸 Moderate      | ✅ CPU preferred        | 🔸 Moderate       | 🔸 Moderate       | Each step may use iterative solvers; memory builds with history data.                                           |
| Batch Pre/Post-Processing | 🔹 Low           | ✅🔴 CPU or GPU capable | 🔹 High           | 🔸 Low–Moderate   | Lightweight tasks like data cleaning, often highly parallel and not memory-intensive.                           |

:::

By first identifying *what* you are doing and then understanding *how* it scales, you can

* Choose appropriate DesignSafe resources
* Avoid over- or under-provisioning compute
* Build workflows that remain efficient as research complexity grows

The goal is not to prescribe a single workflow, but to give you a framework for building flexible, reusable, and scalable computational workflows that evolve alongside your research.

This separation emphasizes two points.

* Users can first identify the type of analysis they are doing without worrying about compute details, establishing conceptual clarity.
* Some tasks are trivially parallel (Monte Carlo, sweeps), while others are memory-bound (coupled simulations) or GPU-accelerated (ML training), reflecting computational diversity.
* DesignSafe provides access to HPC resources that are both scalable and adaptable because there is no one-size-fits-all solution. Some workloads benefit from spreading across many nodes (parallel Monte Carlo), while others require large memory per node (coupled multiphysics). You cannot simply add more nodes as a simulation grows in scope. Different analysis types demand different strategies for scaling.


**How to Use The Tables**

Use the tables to design your computational workflow.

* If your workload looks like Monte Carlo or parametric sweeps, use a job array, since these are *embarrassingly parallel*.
* If you are doing a stepwise time simulation, expect moderate scaling and prefer CPU jobs with sufficient walltime.
* If you are combining multiple solvers (coupled simulation), prioritize memory per node and consider domain decomposition.
* If you are doing ML training or pre/post-processing, these may use different DesignSafe resources (Python/GPU environments, batch preprocessing tools).


## Workloads and Tools Are Distinct

This is a common source of confusion.

A workload is not a tool or platform.

The same workload may be run

* interactively in JupyterHub,
* as a batch job on an HPC system,
* or at scale through a Tapis application.

The workload stays the same. Only the execution strategy changes.


## Mapping Workloads to Execution Strategies

* Monte Carlo / Parametric Sweeps
  Job arrays, many small independent tasks, minimal coordination

* Stepwise Simulations
  Fewer jobs, longer runtimes, stable memory, checkpointing matters

* Coupled Simulations
  Memory-first design, fewer nodes, careful data exchange

* ML Training / Batch Processing
  GPUs or accelerated nodes, fast I/O, preprocessing often dominates runtime



> Scaling a workload is not just about adding more resources. It is about matching the *shape* of the workload to the *structure* of the system.
