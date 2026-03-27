# Storage and File Management

Research data and compute hardware are co-located at [TACC](https://www.tacc.utexas.edu/). A ground-motion database in CommunityData can be referenced directly from a simulation job without downloading it to a laptop and re-uploading it. This co-location is one of [DesignSafe](https://designsafe-ci.org)'s most important advantages, but it means understanding where files live and how they move between environments.

## Storage areas

| Storage Area | Backed Up | Accessible From | Best For |
|---|---|---|---|
| **MyData** | Yes | Data Depot, JupyterHub, VMs, Tapis | Personal files: scripts, inputs, outputs |
| **MyProjects** | Yes | Data Depot, JupyterHub, VMs, Tapis | Team collaboration, curation, publication |
| **CommunityData** | Yes | Data Depot, JupyterHub, VMs, Tapis | Public shared datasets (read-only) |
| **Published** | Yes | Data Depot, JupyterHub, VMs, Tapis | Archived datasets with DOIs (read-only) |
| **Work** | No | Compute nodes, JupyterHub, Data Depot | Active HPC job I/O, staging large inputs |
| **Scratch** | No (purged) | Compute nodes only | Temporary high-speed storage during jobs |

MyData, MyProjects, CommunityData, and Published all live on **Corral**, TACC's networked storage with automatic backups. This is the long-term home for research data. Performance is moderate because access goes over the network.

**Work** and **Scratch** live on [Lustre](https://www.lustre.org/), a parallel filesystem that stripes files across many disks simultaneously. This makes large reads and writes significantly faster than Corral. Work and Scratch are **not backed up**. Use them for staging large inputs and holding outputs temporarily. Always copy important results back to MyData or MyProjects. The performance difference is especially noticeable for jobs that read or write many files, or that perform frequent I/O during execution.

**Node-local storage** (`/tmp`) on each compute node is the fastest option but files disappear when the job ends. Use it for scratch I/O during computation. See [Running HPC Jobs](job-resources.md#node-local-storage) for details on `/tmp` sizes and usage patterns.

```
Prepare in Corral (MyData/MyProjects)
    → Stage to Work for large datasets
    → Run jobs (use /tmp for scratch I/O)
    → Archive results back to Corral
```

## Paths across environments

The same storage area appears at different paths depending on the environment.

### JupyterHub paths

| Storage Area | Path |
|---|---|
| MyData | `/home/jupyter/MyData/` |
| MyProjects | `/home/jupyter/MyProjects/PRJ-XXXX/` |
| Work (Stampede3) | `/home/jupyter/Work/stampede3/` |
| CommunityData | `/home/jupyter/CommunityData/` |
| NHERI Published | `/home/jupyter/NHERI-Published/PRJ-XXXX/` |

### HPC system paths

Each TACC system has its own `$HOME` and `$SCRATCH` filesystems. Only `$WORK` (the Stockyard global shared filesystem) is accessible across systems. The `$WORK` path includes the system name as a subdirectory.

Always use the environment variables (`$HOME`, `$WORK`, `$SCRATCH`) rather than hardcoded paths, since the underlying mount points can change. The examples below show typical paths, but `echo $WORK` will always give the correct current path.

| System | Storage Area | Typical Path | Environment Variable |
|---|---|---|---|
| Stampede3 | Home | `/home1/<groupid>/<username>/` | `$HOME` |
| Stampede3 | Work | `/work/<groupid>/<username>/stampede3/` | `$WORK` |
| Stampede3 | Scratch | `/scratch/<groupid>/<username>/` | `$SCRATCH` |
| Frontera | Home | `/home1/<groupid>/<username>/` | `$HOME` |
| Frontera | Work | `/work/<groupid>/<username>/frontera/` | `$WORK` |
| Frontera | Scratch | use `$SCRATCH` (mount point varies) | `$SCRATCH` |
| Lonestar6 | Home | `/home1/<groupid>/<username>/` | `$HOME` |
| Lonestar6 | Work | `/work/<groupid>/<username>/ls6/` | `$WORK` |
| Lonestar6 | Scratch | `/scratch/<groupid>/<username>/` | `$SCRATCH` |

### Tapis job directory

When [Tapis](https://tapis.readthedocs.io/en/latest/) runs a job, all input files are staged into a single working directory on the compute system, available as `$TAPIS_JOB_WORKDIR`. Every compute node in a multi-node job can see the same staged files through the shared parallel filesystem — inputs are not copied separately to each node.

### dapi path translation

[dapi](https://designsafe-ci.github.io/dapi/) handles path translation automatically. Use DesignSafe paths (as seen in the Data Depot) and let dapi convert them to Tapis URIs:

```python
from dapi import DSClient
ds = DSClient()

# Convert a DesignSafe path to a Tapis URI for job submission
input_uri = ds.files.to_uri("/MyData/opensees/site-response/")

# Convert back
path = ds.files.to_path(input_uri)
```

Common path mappings:

| DesignSafe Path | Tapis URI |
|---|---|
| `/MyData/folder/` | `tapis://designsafe.storage.default/username/folder/` |
| `/MyProjects/PRJ-XXXX/folder/` | `tapis://project-XXXX/folder/` |
| `/CommunityData/folder/` | `tapis://designsafe.storage.community/folder/` |

## File operations with dapi

```python
ds.files.list("/MyData/results/")
ds.files.upload("/MyData/inputs/", "local_file.csv")
ds.files.download("/MyData/results/output.csv", "local_output.csv")
```

## File staging and transfer

When a job is submitted, Tapis automatically stages input files to the execution system before the job starts and archives output back to DesignSafe storage after completion. There is no manual file transfer step.

**Bundle small files.** A directory with 1,000 small CSV files transfers much slower than a single `tar.gz` archive. Bundle inputs before staging.

**Keep shared data in Work.** If multiple jobs reuse the same input data (e.g., 500 ground-motion records for a fragility study), keep it in Work to avoid re-staging for every submission.

**Avoid running against Corral.** Large datasets benefit from the higher I/O bandwidth of Work and Scratch. Running jobs directly against MyData (Corral) is slower and not recommended for production simulations.

For transferring data to and from DesignSafe using Globus, Cyberduck, or command-line tools (scp/rsync), see the [DesignSafe Data Transfer Guide](https://designsafe-ci.org/user-guide/managingdata/datatransfer/).
