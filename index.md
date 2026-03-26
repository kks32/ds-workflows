# DesignSafe Workflows

[![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![DesignSafe](https://img.shields.io/badge/DesignSafe-CI-blue)](https://designsafe-ci.org)
[![Authors](https://img.shields.io/badge/Authors-DesignSafe--CI-orange)](AUTHORS.md)

A guide to running computational workflows on [DesignSafe](https://designsafe-ci.org), from interactive exploration in a Jupyter notebook to production-scale simulations on [TACC](https://www.tacc.utexas.edu/) supercomputers.

## Contents

- [How It Works](guide/how-it-works.md) The DesignSafe portal, compute environments, storage, and workflow design
- [Compute Environments](guide/compute-environments.md) JupyterHub, VMs, HPC systems, queues, and allocations
- [Storage and File Management](guide/storage.md) Storage areas, paths across environments, file staging, and dapi file operations
- [Running HPC Jobs](guide/job-resources.md) Job submission, resource parameters, queues, and file staging
- [Debugging HPC Jobs](guide/debugging.md) Parallel execution, job states, output files, and common failures
- [Parameter Sweeps](guide/parameter-sweeps.md) Running hundreds of independent simulations with PyLauncher
- [DesignSafe Applications](apps/overview.md) Catalog of 45+ available tools
- [Advanced Topics](advanced/tapis.md) Tapis internals, execution strategies, and custom app development

## Quick example

Submit and monitor an HPC job from a Jupyter notebook using [dapi](https://designsafe-ci.github.io/dapi/):

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
