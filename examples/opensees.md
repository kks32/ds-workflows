# OpenSees Structural Analysis

This workflow runs an OpenSees MPI parallel analysis on DesignSafe, from authentication through post-processing of results. The example assumes you have a Tcl model directory uploaded to your MyData storage on DesignSafe.

## Initialize the Client

```python
from dapi import DSClient

ds = DSClient()
```

If credentials are not configured through environment variables or a `.env` file, you will be prompted interactively. See the [authentication guide](../getting-started/authentication.md) for setup options.

## Find the OpenSees App

DesignSafe offers several OpenSees variants. For MPI parallel analysis, use `opensees-mp-s3`.

```python
ds.apps.find("opensees")
```

This prints all OpenSees app variants with their IDs and descriptions. The MP variant runs on Stampede3 and distributes work across multiple processors using MPI. For serial work, `opensees-express` runs on a VM with no queue wait.

To inspect the default resource settings for the app, pull the full details.

```python
app = ds.apps.get_details("opensees-mp-s3")
print(f"Execution system: {app.jobAttributes.execSystemId}")
print(f"Default max minutes: {app.jobAttributes.maxMinutes}")
print(f"Default cores per node: {app.jobAttributes.coresPerNode}")
```

## Prepare the Input Directory

Your Tcl script and any supporting files (ground motion records, material definitions, geometry data) should live in a single directory on DesignSafe Data Depot.

Convert that path to the Tapis URI format that the job system expects. The `verify_exists=True` flag confirms the remote directory is accessible before you proceed, catching path typos early.

```python
input_uri = ds.files.to_uri("/MyData/opensees/bridge_model/", verify_exists=True)
```

## Generate the Job Request

`ds.jobs.generate()` builds a complete Tapis job definition from a few parameters. You specify the app, input location, script filename, resource requirements, and your TACC allocation.

```python
job_request = ds.jobs.generate(
    app_id="opensees-mp-s3",
    input_dir_uri=input_uri,
    script_filename="model.tcl",
    node_count=2,
    cores_per_node=48,
    max_minutes=120,
    allocation="your_allocation",
    job_name="bridge_dynamic_analysis",
)
```

The `node_count` and `cores_per_node` determine the total number of MPI ranks. Two nodes with 48 cores each gives 96 ranks. Match this to your domain decomposition.

This returns a plain dictionary. You can inspect it before submitting and modify any fields directly if needed.

```python
print(job_request)

# Override a field if needed
job_request["maxMinutes"] = 180
```

## Submit the Job

```python
job = ds.jobs.submit(job_request)
print(f"Job submitted: {job.uuid}")
```

The job enters the TACC queue. Depending on system load, it may wait minutes to hours before execution begins.

## Monitor Progress

`job.monitor()` polls the job status at a regular interval and prints updates as the job moves through queue stages.

```python
final_status = job.monitor(interval=15, timeout_minutes=240)
ds.jobs.interpret_status(final_status, job.uuid)
```

The call blocks until the job reaches a terminal state (FINISHED, FAILED, or CANCELLED). If you lose your notebook session, you can reconnect later using the job UUID.

```python
from dapi.jobs import SubmittedJob

job = SubmittedJob(ds._tapis, "your-job-uuid")
job.monitor()
```

## Check Results

Print a summary of the runtime, including start and end times.

```python
job.print_runtime_summary()
```

List the output files produced by the job.

```python
outputs = job.list_outputs()
for f in outputs:
    print(f"{f.name} ({f.type}, {f.size} bytes)")
```

Read the main log to confirm the analysis completed without errors.

```python
stdout = job.get_output_content("tapisjob.out")
print(stdout)
```

If something went wrong, check the error log.

```python
stderr = job.get_output_content("tapisjob.err", missing_ok=True)
if stderr:
    print(stderr)
```

## Access Archived Results

After the job finishes, Tapis archives all output files back to your DesignSafe storage. You can list files in subdirectories of the archive and download specific results.

```python
# List files in the results subdirectory
results = job.list_outputs(path="results/")

# Download a specific output file to your local machine
job.download_output("results/disp_history.out", local_path="./disp_history.out")
```

## Post-Processing Response Spectra

Once you have the displacement or acceleration output from OpenSees, you can compute response spectra in Python. This example reads an acceleration time history, computes the pseudo-acceleration response spectrum via FFT, and plots the result.

```python
import numpy as np
import matplotlib.pyplot as plt

# Read the acceleration history from the archived output
# Assumes two columns, space-delimited, one timestep per row
data = np.loadtxt("./disp_history.out")
time_vec = data[:, 0]
accel = data[:, 1]

dt = time_vec[1] - time_vec[0]
nstep = len(accel)
total_time = nstep * dt
```

Compute the response spectrum using the frequency-domain transfer function approach. This method applies a single-degree-of-freedom oscillator at each target period.

```python
def response_spectrum(accel, dt, damping=0.05, n_periods=100):
    """Compute pseudo-acceleration response spectrum via FFT."""
    a = np.insert(accel, 0, 0.0)
    nstep = len(a) - 1
    total_time = nstep * dt

    periods = np.logspace(-3.0, 1.0, n_periods)
    dw = 2.0 * np.pi / total_time
    w = np.arange(0, (nstep + 1) * dw, dw)
    afft = np.fft.fft(a)

    k = 1000.0
    sa = np.zeros(n_periods)

    for j in range(n_periods):
        m = ((periods[j] / (2 * np.pi)) ** 2) * k
        c = 2 * damping * np.sqrt(k * m)
        h = np.zeros(nstep + 2, dtype=complex)

        for l in range(int(nstep / 2 + 1)):
            h[l] = 1.0 / (-m * w[l] ** 2 + 1j * c * w[l] + k)
            h[nstep + 1 - l] = np.conj(h[l])

        qfft = -m * afft
        u = np.array([h[l] * qfft[l] for l in range(nstep + 1)])
        utime = np.real(np.fft.ifft(u))

        sd = np.max(np.abs(utime))
        sa[j] = (2 * np.pi / periods[j]) ** 2 * sd

    return periods, sa


periods, sa = response_spectrum(accel, dt)
```

Plot the result.

```python
plt.figure(figsize=(10, 6))
plt.loglog(periods, sa)
plt.xlabel("Period (s)")
plt.ylabel("Pseudo-Acceleration (g)")
plt.title("5% Damped Response Spectrum")
plt.grid(True, which="both", alpha=0.3)
plt.tight_layout()
plt.savefig("response_spectrum.png", dpi=150)
plt.show()
```

## Complete Workflow

Here is the full submission-to-results workflow in one block.

```python
from dapi import DSClient

ds = DSClient()

# Prepare input
input_uri = ds.files.to_uri("/MyData/opensees/bridge_model/", verify_exists=True)

# Generate and submit
job_request = ds.jobs.generate(
    app_id="opensees-mp-s3",
    input_dir_uri=input_uri,
    script_filename="model.tcl",
    node_count=2,
    cores_per_node=48,
    max_minutes=120,
    allocation="your_allocation",
)
job = ds.jobs.submit(job_request)

# Monitor
final_status = job.monitor()
ds.jobs.interpret_status(final_status, job.uuid)

# Results
job.print_runtime_summary()
job.list_outputs()
print(job.get_output_content("tapisjob.out"))

# Download output for local post-processing
job.download_output("results/disp_history.out", local_path="./disp_history.out")
```

## Related

- [OpenSees on DesignSafe](../designsafe-apps/opensees.md) for app variants and what the app automates
- [SLURM MPI](../hpc/mpi.md) for parallel execution concepts, rank management, and file I/O strategies
