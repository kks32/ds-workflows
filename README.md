# DesignSafe Workflows

[![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![DesignSafe](https://img.shields.io/badge/DesignSafe-CI-blue)](https://designsafe-ci.org)
[![Authors](https://img.shields.io/badge/Authors-DesignSafe--CI-orange)](AUTHORS.md)

A guide to running computational workflows on [DesignSafe](https://designsafe-ci.org) using [dapi](https://designsafe-ci.github.io/dapi/), [Tapis](https://tapis.readthedocs.io/en/latest/), and [TACC](https://www.tacc.utexas.edu/) high-performance computing systems.

## What this covers

- **How it works** including the portal, compute environments (JupyterHub, VMs, HPC), storage systems, and workflow design
- **Running HPC jobs** with dapi, including resource selection, queues, and allocations
- **Debugging** failed jobs using Tapis job states and SLURM output files
- **Parallel computing** with MPI for multi-node simulations
- **Parameter sweeps** with PyLauncher for high-throughput studies
- **DesignSafe applications** catalog with 45+ tools across simulation, analysis, visualization, GIS, and hazard data
- **Advanced topics** including Tapis internals, execution strategies, and custom app development

## Quick start

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

## Building locally

```bash
npm install mystmd
npx myst start
```

The site will be available at `http://localhost:3000`.

## License

This work is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). See [LICENSE.md](LICENSE.md).

## Authors

See [AUTHORS.md](AUTHORS.md).
