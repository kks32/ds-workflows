# DesignSafe Workflows

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

- [How DesignSafe Runs Your Jobs](guide/how-it-works.md)
- [Compute Environments](guide/compute-environments.md)
- [Storage and File Management](guide/storage.md)
- [Job Resources](guide/job-resources.md)
- [Debugging Failed Jobs](guide/debugging.md)
- [Parallel Computing](guide/parallel-computing.md)
- [Parameter Sweeps](guide/parameter-sweeps.md)
- [DesignSafe Applications](apps/overview.md)
- [Advanced Topics](advanced/tapis.md)
