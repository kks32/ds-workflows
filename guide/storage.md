# Storage and File Management

[How It Works](how-it-works.md) introduced the six storage areas on DesignSafe. This page covers the practical details of paths, file movement, and transfer strategies.

## What is backed up and what is not

MyData, MyProjects, CommunityData, and Published all live on Corral, [TACC](https://www.tacc.utexas.edu/)'s backed-up networked storage system. Work and Scratch are NOT backed up. Files lost there cannot be recovered. Always copy important results back to MyData or MyProjects after jobs complete.

## Path Formats Across Environments

The same storage area appears at different paths depending on where it is accessed.

JupyterHub paths (mounted in the notebook file browser)

| Storage Area | Path |
|---|---|
| MyData | `/home/jupyter/MyData/` |
| MyProjects | `/home/jupyter/MyProjects/PRJ-...` |
| Work | `/home/jupyter/Work/stampede3/` |
| CommunityData | `/home/jupyter/CommunityData/` |
| NHERI Published | `/home/jupyter/NHERI-Published/PRJ-...` |

[Stampede3](https://docs.tacc.utexas.edu/hpc/stampede3/) paths (absolute UNIX paths on the HPC system)

| Storage Area | Path |
|---|---|
| Home | `/home1/<groupid>/<username>/` |
| Work | `/work2/<groupid>/<username>/stampede3/` |
| Scratch | `/scratch/<groupid>/<username>/` |

Actual paths on Stampede3 can be verified with `echo $HOME`, `echo $WORK`, and `echo $SCRATCH`.

## How dapi Translates Paths

When submitting jobs through [dapi](https://designsafe-ci.github.io/dapi/), a single call converts [DesignSafe](https://designsafe-ci.org) paths to [Tapis](https://tapis.readthedocs.io/en/latest/) URIs.

```python
ds.files.to_uri("/MyData/analysis/")
```

This converts a familiar DesignSafe path into the Tapis URI that the job submission system requires. See the [dapi documentation](https://designsafe-ci.github.io/dapi/) for the full file-management API.

## The File Movement Workflow

Data moves through a predictable cycle during job execution.

1. Prepare input files in MyData or MyProjects.
2. Submit the job. Tapis stages inputs from DesignSafe storage to the execution system's Work or Scratch directory.
3. The application reads from and writes to local high-performance storage during execution.
4. After the job completes, Tapis archives results back to DesignSafe storage.

There is no need to manually transfer files between DesignSafe and TACC systems. Tapis handles staging and archiving automatically.

## File Transfer Tips

Many small files transfer slowly. A directory with 1,000 small CSV files takes significantly longer to stage than a single tar.gz archive containing the same data. Bundle input files before staging whenever possible to reduce overhead.

If multiple jobs share the same input data (for example, 500 ground-motion records reused across a fragility study), keep that data in Work to avoid re-staging it for every submission.

Archive only final outputs. Intermediate files and logs can remain in Work or Scratch temporarily, then be discarded once the important results have been copied.

Large datasets benefit from the higher I/O bandwidth of Work and Scratch. Running jobs directly against Corral (MyData) is slower and not recommended for production simulations.

[Running HPC Jobs](job-resources.md) covers resource selection and the full job submission workflow.
