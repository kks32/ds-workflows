# DesignSafe Workflows

[![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![DesignSafe](https://img.shields.io/badge/DesignSafe-CI-blue)](https://designsafe-ci.org)
[![Authors](https://img.shields.io/badge/Authors-DesignSafe--CI-orange)](AUTHORS.md)

Run simulations on [TACC](https://www.tacc.utexas.edu/) supercomputers from a [DesignSafe](https://designsafe-ci.org) Jupyter notebook with [dapi](https://designsafe-ci.github.io/dapi/).

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

For installation, authentication, and API reference, see the [dapi documentation](https://designsafe-ci.github.io/dapi/).

## Contents

- [How It Works](guide/how-it-works.md) (portal, environments, storage, workflow design)
- [Running HPC Jobs](guide/job-resources.md)
- [Debugging Failed Jobs](guide/debugging.md)
- [Parallel Computing](guide/parallel-computing.md)
- [Parameter Sweeps](guide/parameter-sweeps.md)
- [DesignSafe Applications](apps/overview.md)
- [Advanced Topics](advanced/tapis.md)
