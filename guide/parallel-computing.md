# Parallel Computing

A single CPU core executes instructions one at a time. For a small model, that is fast enough. But a bridge with millions of finite elements, a coastal surge simulation spanning hundreds of kilometers, or turbulent flow through complex geometry can take days on one core. Parallel computing splits the problem so many cores work on it at the same time.

Many [DesignSafe](https://designsafe-ci.org) users run serial jobs or parameter sweeps and never touch parallel computing directly. A researcher submitting 500 independent OpenSeesPy analyses through [PyLauncher](https://github.com/TACC/pylauncher) is running serial tasks in parallel, not using MPI. A MATLAB script processing data files one after another needs only one core. This page is for workflows where a single simulation must be divided across multiple cores or nodes because it is too large or too slow for one.

## MPI in Plain Terms

[MPI](https://www.mpi-forum.org/) (Message Passing Interface) is a way for separate processes to exchange data while working on different parts of the same problem. Each process has its own memory and runs independently. When one process needs data that another process computed (for example, the forces at a shared boundary between two mesh partitions), MPI handles the exchange.

On [TACC](https://www.tacc.utexas.edu/) systems, MPI is the default model for parallel execution. Several DesignSafe applications use it internally.

| Application | Parallelism | Typical Use |
|---|---|---|
| OpenSees MP | MPI | Structural analysis with domain decomposition |
| [ADCIRC](https://adcirc.org/) | MPI | Coastal and storm surge modeling |
| [OpenFOAM](https://www.openfoam.com/) | MPI | Computational fluid dynamics |
| MPM | MPI | Material point method simulations |
| OpenSees (serial variants) | Serial (no MPI) | Single-process structural analysis on VM or HPC |
| Agnostic App | Serial (no MPI) | General-purpose Python, OpenSeesPy, PyLauncher |
| [MATLAB](https://www.mathworks.com/products/matlab.html) | Serial (no MPI) | Data analysis and algorithms |

Researchers using MPI applications do not write MPI code themselves. The application handles the parallelism internally. The researcher's role is to request the right number of nodes and cores and ensure the model is set up for the intended number of parallel processes.

## Submitting a parallel job

A parallel job looks the same as any other dapi submission, with `node_count` and `cores_per_node` set to match the problem. Here is an [OpenSees](https://opensees.berkeley.edu/) MP example that runs a structural model across 96 cores (2 nodes, 48 cores each):

```python
from dapi import DSClient

ds = DSClient()
input_uri = ds.files.to_uri("/MyData/opensees/bridge-model/")

job_request = ds.jobs.generate(
    app_id="opensees-mp-s3",
    input_dir_uri=input_uri,
    script_filename="analysis.tcl",
    node_count=2,
    cores_per_node=48,
    max_minutes=120,
    allocation="your_allocation",
    queue="skx",
)

job = ds.jobs.submit(job_request)
job.monitor()
```

This produces 96 MPI ranks (2 x 48). The OpenSees model must be set up for 96 parallel processes. If the number of ranks does not match the model's domain decomposition, the simulation will either fail or produce incorrect results.

| dapi parameter | Effect on MPI |
|---|---|
| `node_count` | Number of physical machines |
| `cores_per_node` | MPI ranks per machine |
| Total ranks | node_count x cores_per_node (must match the model's decomposition) |

## Ranks

When an MPI program starts, [SLURM](https://slurm.schedmd.com/documentation.html) launches multiple copies of it. Each copy is called a **rank** and receives a unique integer ID from 0 through N-1. The total number of ranks equals `node_count x cores_per_node`.

A concrete example. Requesting 2 nodes with 48 cores each produces 96 MPI ranks. Each rank handles part of the computational domain. Rank 0 might process the left portion of a structural mesh, rank 47 the right portion on the first node, and ranks 48 through 95 cover the second node.

Every rank executes the same program, but each operates on different data. Ranks cooperate by exchanging messages through the MPI library.

## SLURM Environment Variables

SLURM sets several environment variables that a program can read at runtime to determine its identity within the parallel job.

| Variable | Meaning |
|---|---|
| `SLURM_PROCID` | Global MPI rank ID (0 through N-1) |
| `SLURM_LOCALID` | Rank index within the current node |
| `SLURM_NODEID` | Node index within the job allocation |
| `SLURM_JOB_ID` | Scheduler job identifier |

Rank 12 might have `SLURM_NODEID=0` (first node) and `SLURM_LOCALID=12` (thirteenth rank on that node). Rank 60 might have `SLURM_NODEID=1` and `SLURM_LOCALID=12`, occupying the same local position on the second node.

## MPI vs [OpenMP](https://www.openmp.org/)

Two models exist for parallel execution on HPC systems. They differ in how processes share memory and communicate.

| Feature | MPI (Multi-Processing) | OpenMP (Multi-Threading) |
|---|---|---|
| Execution model | Separate processes, each with its own memory | Threads within a single process, sharing memory |
| Communication | Message passing between ranks | Shared memory access between threads |
| Default on TACC | Yes. One MPI process per physical core. | Supported but not enabled by default. |
| Setup | No special configuration needed | Requires setting `OMP_NUM_THREADS` and adjusting the job script |
| Best fit | Distributed tasks across nodes | Fine-grained shared-memory parallelism within one node |

MPI is the default execution model on [Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/) and other TACC systems. The `ibrun` launcher places one MPI process per physical core. To use OpenMP (or a hybrid MPI + OpenMP approach), set the thread count explicitly and request fewer MPI processes to avoid oversubscribing cores.

## Rank-Aware File Management

Two ranks writing to the same filename will overwrite each other. This produces corrupted output, intermittent failures, or silently wrong results.

Use the rank ID in output file paths to prevent collisions.

```python
import os

rank = os.environ.get("SLURM_PROCID", "0")
output_file = f"results_rank{rank}.txt"
```

All ranks on the same node share the same `/tmp` directory, so even temporary files need rank-based naming to avoid conflicts.

## When Parallel Computing Is Not Needed

Not every workflow benefits from MPI.

PyLauncher sweeps run many independent serial tasks inside a single SLURM allocation. Each task is a separate process that does not communicate with the others. PyLauncher handles the dispatching. A fragility study with 500 ground-motion records, each processed by a standalone OpenSeesPy script, is a perfect fit for PyLauncher and has nothing to do with MPI.

Single serial scripts that process data sequentially need only one core. Requesting more cores does not speed them up.

Small models that fit on one node and complete in a reasonable time gain nothing from MPI overhead. Running on one core or a small number of cores within a single node is simpler and avoids the complexity of domain decomposition.

If a workload consists of many independent runs rather than one large coupled simulation, a [parameter sweep](parameter-sweeps.md) with PyLauncher or [dapi](https://designsafe-ci.github.io/dapi/) is a better fit than MPI parallelism.
