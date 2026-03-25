# MPM Simulation

This workflow runs a Material Point Method simulation on DesignSafe, covering app discovery, resource configuration, job submission, and output inspection. MPM simulations model large-deformation problems in geotechnical engineering (landslides, slope failures, soil-structure interaction) by tracking material behavior through particles that move across a background mesh.

## Initialize the Client

```python
from dapi import DSClient

ds = DSClient()
```

## Find the MPM App

Search for available MPM applications on DesignSafe.

```python
ds.apps.find("mpm")
```

The primary app is `mpm-s3`, which runs on Stampede3. Pull its full details to understand the default resource configuration.

```python
app = ds.apps.get_details("mpm-s3")
print(f"Execution system: {app.jobAttributes.execSystemId}")
print(f"Default nodes: {app.jobAttributes.nodeCount}")
print(f"Default cores per node: {app.jobAttributes.coresPerNode}")
print(f"Default max minutes: {app.jobAttributes.maxMinutes}")
```

These defaults may be conservative or generous depending on your problem size. Override them when generating your job request.

## Prepare the Input Directory

Place your MPM JSON configuration file, mesh files, and any material property files in a single directory on DesignSafe. The directory should contain everything the solver needs to run.

```python
input_uri = ds.files.to_uri("/MyData/mpm/slope_stability/", verify_exists=True)
```

The `verify_exists=True` flag confirms the remote path is accessible before you proceed.

## Resource Guidelines

MPM simulations scale with particle count. The table below gives starting-point recommendations for Stampede3. Actual requirements depend on the constitutive model complexity, number of output steps, and mesh resolution.

| Problem Size | Nodes | Cores | Memory (MB) | Max Minutes |
|---|---|---|---|---|
| Small (under 100K particles) | 1 | 56 | 192000 | 60 |
| Medium (100K to 1M) | 2 | 112 | 384000 | 240 |
| Large (over 1M) | 4+ | 224+ | 768000+ | 480 |

For a first run, start with the small configuration and a short time limit. You can inspect the output to estimate how long the full simulation will take, then scale up.

## Generate the Job Request

```python
job_request = ds.jobs.generate(
    app_id="mpm-s3",
    input_dir_uri=input_uri,
    script_filename="mpm_input.json",
    node_count=2,
    cores_per_node=56,
    max_minutes=240,
    memory_mb=384000,
    allocation="your_allocation",
    job_name="slope_stability_001",
)
```

The `script_filename` is your MPM input JSON file. The solver reads this at startup to configure the problem geometry, materials, boundary conditions, and output settings.

You can inspect the generated request before submitting.

```python
print(job_request)
```

## Submit the Job

```python
job = ds.jobs.submit(job_request)
print(f"Job submitted: {job.uuid}")
```

## Monitor Progress

MPM simulations can run for several hours. Set a generous timeout for monitoring.

```python
final_status = job.monitor(interval=30, timeout_minutes=300)
ds.jobs.interpret_status(final_status, job.uuid)
```

The monitor polls every 30 seconds and prints status changes. If you close your notebook, reconnect later with the job UUID.

```python
from dapi.jobs import SubmittedJob

job = SubmittedJob(ds._tapis, "your-job-uuid")
status = job.get_status()
print(status)
```

## Check the Runtime Summary

```python
job.print_runtime_summary()
```

This prints start time, end time, and total duration. For long-running simulations, this helps you calibrate future time limits.

## List and Read Output Files

```python
outputs = job.list_outputs()
for f in outputs:
    print(f"{f.name} ({f.type}, {f.size} bytes)")
```

Read the standard output log to confirm the solver completed successfully.

```python
stdout = job.get_output_content("tapisjob.out")
print(stdout)
```

Check stderr for any warnings or errors the solver reported.

```python
stderr = job.get_output_content("tapisjob.err", missing_ok=True)
if stderr:
    print(stderr)
```

## Common MPM Output Files

| File | Contents |
|---|---|
| `particles*.vtp` | Particle positions, velocities, stresses at each output step |
| `mesh*.vtu` | Background mesh state |
| `energy.csv` | Kinetic and strain energy over time |
| `mpm_input.json` | Copy of the input configuration |

The VTP and VTU files can be visualized in ParaView. The energy CSV tracks conservation and convergence over the simulation timeline.

## Access Archived Results

After the job finishes, download files for local post-processing.

```python
# Download the energy history
job.download_output("energy.csv", local_path="./energy.csv")

# Download a specific particle output file
job.download_output("particles_100.vtp", local_path="./particles_100.vtp")
```

For a quick look at the energy evolution without downloading files, read the content directly.

```python
import pandas as pd
from io import StringIO

energy_content = job.get_output_content("energy.csv")
energy = pd.read_csv(StringIO(energy_content))
print(energy.tail(10))
```

## Complete Workflow

```python
from dapi import DSClient

ds = DSClient()

# Find and inspect the app
app = ds.apps.get_details("mpm-s3")

# Prepare input
input_uri = ds.files.to_uri("/MyData/mpm/slope_stability/", verify_exists=True)

# Generate and submit
job_request = ds.jobs.generate(
    app_id="mpm-s3",
    input_dir_uri=input_uri,
    script_filename="mpm_input.json",
    node_count=2,
    cores_per_node=56,
    max_minutes=240,
    memory_mb=384000,
    allocation="your_allocation",
)
job = ds.jobs.submit(job_request)

# Monitor
final_status = job.monitor(interval=30, timeout_minutes=300)
ds.jobs.interpret_status(final_status, job.uuid)

# Results
job.print_runtime_summary()
job.list_outputs()
print(job.get_output_content("tapisjob.out"))
```
