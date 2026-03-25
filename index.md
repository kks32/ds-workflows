# Computational Workflows on DesignSafe

This guide covers how to run computational workflows on [DesignSafe](https://designsafe-ci.org) using [dapi](https://designsafe-ci.github.io/dapi/), [Tapis](https://tapis.readthedocs.io/en/latest/), and [TACC](https://www.tacc.utexas.edu/) high-performance computing systems. It walks through the infrastructure, concepts, and application-specific details needed to move from a working script to production-scale HPC computation.

```python
from dapi import DSClient

ds = DSClient()

job_request = ds.jobs.generate(
    app_id="opensees-mp-s3",
    input_dir_uri="/MyData/analysis/input/",
    script_filename="model.tcl",
    max_minutes=60,
    allocation="your_allocation",
)

job = ds.jobs.submit(job_request)
job.monitor()
```

For dapi installation, authentication, and API reference, see the [dapi documentation](https://designsafe-ci.github.io/dapi/).

## Contents

- [Your First Job](getting-started/first-job.md)
- [Workflows](workflows/overview.md) on architecture, design, and execution strategies
- [HPC](hpc/overview.md) on SLURM job scheduling, scripts, resources, MPI, and PyLauncher
- [Tapis](tapis/overview.md) on middleware automation, job lifecycle, and app development
- [DesignSafe Apps](designsafe-apps/overview.md) including OpenSees, OpenFOAM, ADCIRC, and the Agnostic App
- [Examples](examples/opensees.md) with complete workflows for specific applications
