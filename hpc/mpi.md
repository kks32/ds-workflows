# Ranks in MPI Jobs

## MPI Ranks, Nodes, and Scratch Space

Before using node-local `/tmp` storage safely in an MPI job, you need to understand how parallel processes are organized and identified at runtime. The file-management patterns that follow rely on these concepts to avoid file collisions, race conditions, and data loss.

This section introduces the core MPI and Slurm concepts referenced throughout the storage chapter.

---

## What Is an MPI Rank?

In an MPI job, your program is launched multiple times in parallel. Each running instance is called a rank.

* Every rank executes the same program
* Each rank has a unique integer ID
* Ranks typically cooperate by exchanging data via MPI

The rank ID is commonly referred to as the MPI rank (0, 1, 2, ..., N-1).

```text
MPI rank  (0, 1, 2, ..., N−1)
```

In Slurm-launched jobs (including Tapis jobs on TACC systems), this value is exposed as `SLURM_PROCID`.

```bash
SLURM_PROCID
```

For example, `SLURM_PROCID=0` indicates rank 0 (often used for coordination or aggregation), and `SLURM_PROCID=7` indicates the 8th MPI process.

When writing files, two ranks with the same filename will overwrite each other unless separated. This is why rank-aware paths are essential.

---

## Nodes vs. Ranks

A node is a physical (or virtual) machine in the cluster.

Each node has its own CPUs, its own memory, and its own `/tmp` directory. Multiple MPI ranks typically run on the same node.

Slurm provides two important identifiers in addition to the rank ID.

| Concept    | Meaning                      | Slurm variable  |
| ---------- | ---------------------------- | --------------- |
| Rank ID    | Global MPI process index     | `SLURM_PROCID`  |
| Node ID    | Which node in the allocation | `SLURM_NODEID`  |
| Local rank | Rank index within a node     | `SLURM_LOCALID` |

For example, Rank 12 may have `SLURM_NODEID=1` (second node) and `SLURM_LOCALID=4` (5th rank on that node).

This distinction matters because all ranks on the same node share the same `/tmp` directory.


---

## Common Environment Variables

These variables are automatically defined by Slurm (and therefore by Tapis on Slurm systems).

| Variable            | Description                          |
| ------------------- | ------------------------------------ |
| `SLURM_PROCID`      | Global MPI rank ID                   |
| `SLURM_LOCALID`     | Rank index within a node             |
| `SLURM_NODEID`      | Node index within the job allocation |
| `SLURM_JOB_ID`      | Scheduler job identifier             |
| `USER`              | Unix username                        |
| `TAPIS_JOB_WORKDIR` | Shared job execution directory       |

Throughout this chapter, these variables are used to create unique per-rank or per-node directories, coordinate file copies safely, and control cleanup behavior.

---

## Why Rank-Aware File Management Is Essential

Without rank-aware file paths, ranks overwrite each other's temporary files, partial files are read before being fully written, and jobs fail intermittently in ways that are difficult to debug.

By explicitly separating files by rank and node, you gain deterministic behavior, reproducibility, and safe use of high-performance node-local storage.

---

### How This Fits into the Broader Storage Model

Shared filesystems (e.g., Work, Scratch) provide a single job directory visible to all nodes and are ideal for inputs and final outputs.

Node-local `/tmp` is fast but ephemeral, requires explicit management, and is ideal for temporary, I/O-intensive data.

The MPI-safe patterns that follow build directly on these concepts and show how to combine correctness and performance in large-scale parallel workflows.
