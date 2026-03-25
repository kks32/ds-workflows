# Your First Job

This walkthrough takes you from zero to a completed HPC job on DesignSafe.

## Prerequisites

Install [dapi](https://github.com/DesignSafe-CI/dapi) and set up authentication. The [dapi documentation](https://designsafe-ci.github.io/dapi/) covers installation, credentials, and the full API reference.

```python
pip install dapi
```

## Submit a job

```python
from dapi import DSClient

ds = DSClient()

# Find available applications
ds.apps.find("matlab")

# Translate your Data Depot path to a Tapis URI
input_uri = ds.files.to_uri("/MyData/analysis/input/")

# Generate a job request
job_request = ds.jobs.generate(
    app_id="matlab-24.4.0u2",
    input_dir_uri=input_uri,
    script_filename="run_analysis.m",
    max_minutes=30,
    allocation="your_allocation",
)

# Submit and monitor
job = ds.jobs.submit(job_request)
job.monitor()

# Check results
job.print_runtime_summary()
job.list_outputs()
print(job.get_output_content("tapisjob.out"))
```

`DSClient()` handles authentication and TMS credential setup automatically. If environment variables or a `.env` file are not configured, it prompts interactively.

The `app_id` comes from `ds.apps.find()`. The `script_filename` is the file inside your input directory that the application executes. The `allocation` is your TACC project allocation string.

`job.monitor()` polls the job status and prints updates as the job moves through QUEUED, RUNNING, and FINISHED states. It blocks until the job reaches a terminal state.

## What happened under the hood

When you called `ds.jobs.submit()`, dapi sent your job request to Tapis. Tapis then staged your input files to the execution system, generated a SLURM batch script, submitted it with `sbatch`, and monitored execution. After completion, Tapis archived the output files back to your DesignSafe storage.

The rest of this guide explains each layer of that process in detail.

- [Workflows](../workflows/overview.md) covers how DesignSafe connects interfaces to HPC systems
- [SLURM](../hpc/overview.md) explains the job scheduler that runs your computation
- [Tapis](../tapis/overview.md) describes the middleware that automates SLURM on your behalf
- [Examples](../examples/opensees.md) shows complete workflows for specific applications
