# OpenFOAM CFD Simulation

This workflow runs an OpenFOAM case on DesignSafe, from case setup through extraction and plotting of force coefficients. OpenFOAM handles computational fluid dynamics problems including external aerodynamics, internal flows, and free-surface simulations.

## Initialize the Client

```python
from dapi import DSClient

ds = DSClient()
```

## Find the OpenFOAM App

```python
ds.apps.find("openfoam")
```

This lists available OpenFOAM versions with their app IDs. Pick the version that matches your case files.

Inspect the app defaults to understand the execution environment.

```python
app = ds.apps.get_details("openfoam-9.0")
print(f"Execution system: {app.jobAttributes.execSystemId}")
print(f"Default cores: {app.jobAttributes.coresPerNode}")
```

## Prepare the Case Directory

OpenFOAM expects a standard case directory structure.

```
cylinder_flow/
    0/              # Initial and boundary conditions
    constant/       # Mesh definition and physical properties
    system/         # Solver controls (controlDict, fvSchemes, fvSolution)
    run.sh          # Script that runs mesh generation and the solver
```

The `run.sh` script typically calls blockMesh or snappyHexMesh for meshing, decomposePar for domain decomposition, the solver (pisoFoam, simpleFoam, pimpleFoam, etc.), and reconstructPar to reassemble parallel results. The OpenFOAM app on DesignSafe handles module loading and MPI execution automatically.

Upload this case directory to DesignSafe Data Depot, then convert the path.

```python
input_uri = ds.files.to_uri("/MyData/openfoam/cylinder_flow/", verify_exists=True)
```

## Resource Guidelines

CFD mesh resolution drives the computational cost. These recommendations assume a transient incompressible solver on Stampede3.

| Mesh Size | Nodes | Cores | Max Minutes |
|---|---|---|---|
| Under 1M cells | 1 | 48 | 120 |
| 1M to 10M cells | 2-4 | 96-192 | 360 |
| Over 10M cells | 4+ | 192+ | 720 |

The core count should align with your decomposition. If you decompose into 96 subdomains, request at least 96 cores (2 nodes with 48 cores each).

## Generate and Submit the Job

```python
job_request = ds.jobs.generate(
    app_id="openfoam-9.0",
    input_dir_uri=input_uri,
    script_filename="run.sh",
    node_count=2,
    cores_per_node=48,
    max_minutes=240,
    allocation="your_allocation",
    job_name="cylinder_Re1000",
)

job = ds.jobs.submit(job_request)
print(f"Job submitted: {job.uuid}")
```

The `script_filename` points to your run script, which orchestrates the full simulation pipeline (meshing, decomposition, solving, reconstruction).

You can inspect and modify the job request before submitting.

```python
print(job_request)

# Add extra environment variables if needed
job_request["parameterSet"]["envVariables"].append(
    {"key": "FOAM_SIGFPE", "value": "false"}
)
```

## Monitor Progress

```python
final_status = job.monitor(interval=30, timeout_minutes=400)
ds.jobs.interpret_status(final_status, job.uuid)
```

CFD simulations often run for hours. The monitor prints status changes as the job moves from QUEUED through RUNNING to FINISHED.

## Check Results

```python
job.print_runtime_summary()

outputs = job.list_outputs()
for f in outputs:
    print(f"{f.name} ({f.type})")
```

Read the job log to confirm the solver ran to completion. Look for "End" at the bottom of the OpenFOAM output, which indicates the solver finished its time loop.

```python
stdout = job.get_output_content("tapisjob.out")
print(stdout)
```

## Post-Processing Force Coefficients

If your `controlDict` includes a `forceCoeffs` function object, OpenFOAM writes force coefficient data to the `postProcessing/` directory during the simulation. This data is useful for tracking convergence and extracting aerodynamic loads.

Read the force coefficient file from the archived output.

```python
content = job.get_output_content("postProcessing/forceCoeffs1/0/coefficient.dat")
print(content[:500])  # Preview the first few lines
```

Parse the data into a DataFrame. The file uses a comment header (lines starting with `#`) followed by tab-delimited columns.

```python
import pandas as pd
from io import StringIO

# Filter out comment lines
lines = content.strip().split("\n")
data_lines = [line for line in lines if not line.startswith("#")]
cleaned = "\n".join(data_lines)

df = pd.read_csv(
    StringIO(cleaned),
    sep=r"\s+",
    names=["Time", "Cd", "Cs", "Cl", "CmRoll", "CmPitch", "CmYaw"],
)

print(df.describe())
```

Plot the drag and lift coefficients over time.

```python
import matplotlib.pyplot as plt

fig, (ax1, ax2) = plt.subplots(2, 1, figsize=(10, 6), sharex=True)

ax1.plot(df["Time"], df["Cd"], linewidth=0.8)
ax1.set_ylabel("Drag Coefficient (Cd)")
ax1.grid(True, alpha=0.3)

ax2.plot(df["Time"], df["Cl"], linewidth=0.8, color="tab:orange")
ax2.set_ylabel("Lift Coefficient (Cl)")
ax2.set_xlabel("Time (s)")
ax2.grid(True, alpha=0.3)

fig.suptitle("Force Coefficients over Time")
plt.tight_layout()
plt.savefig("force_coefficients.png", dpi=150)
plt.show()
```

For steady-state simulations, the coefficients should converge to constant values. For transient cases with vortex shedding (like flow around a cylinder), expect periodic oscillations in Cl.

## Download Results for Local Visualization

OpenFOAM time directories and the reconstructed fields are large. Download selectively.

```python
# Download the force coefficient data
job.download_output(
    "postProcessing/forceCoeffs1/0/coefficient.dat",
    local_path="./coefficient.dat",
)

# List available time directories
time_dirs = job.list_outputs()
for f in time_dirs:
    if f.type == "dir":
        print(f.name)
```

For full field visualization, use ParaView. You can download the entire case or mount your DesignSafe storage and open the `.foam` file directly.

## Complete Workflow

```python
from dapi import DSClient

ds = DSClient()

# Prepare input
input_uri = ds.files.to_uri("/MyData/openfoam/cylinder_flow/", verify_exists=True)

# Generate and submit
job_request = ds.jobs.generate(
    app_id="openfoam-9.0",
    input_dir_uri=input_uri,
    script_filename="run.sh",
    node_count=2,
    cores_per_node=48,
    max_minutes=240,
    allocation="your_allocation",
)
job = ds.jobs.submit(job_request)

# Monitor
final_status = job.monitor(interval=30, timeout_minutes=400)
ds.jobs.interpret_status(final_status, job.uuid)

# Results
job.print_runtime_summary()
job.list_outputs()
print(job.get_output_content("tapisjob.out"))
```

## Related

- [OpenFOAM on DesignSafe](../designsafe-apps/openfoam.md) for solver options and what the app automates
