# App Catalog

[DesignSafe](https://designsafe-ci.org) provides access to simulation, analysis, visualization, and data tools for natural hazards research. Applications run on [TACC](https://www.tacc.utexas.edu/) HPC systems, virtual machines, web interfaces, or through [JupyterHub](https://jupyter.designsafe-ci.org/) notebooks. See [How It Works](../guide/how-it-works.md) for details on where each type of application runs.

## Simulation

| Application | Description | Execution | License |
|---|---|---|---|
| [OpenSees](https://opensees.berkeley.edu/) | Structural and geotechnical earthquake simulation (serial, MP, SP, Py variants) | HPC, VM | Open Source |
| [OpenFOAM](https://www.openfoam.com/) | Computational fluid dynamics for flow, heat transfer, turbulence | HPC | Open Source |
| [ADCIRC](https://adcirc.org/) | Coastal circulation and storm surge finite element modeling | HPC | Open Source |
| [LS-DYNA](https://lsdyna.ansys.com/) | Explicit finite element analysis for dynamic problems | HPC | Licensed |
| MPM | Material Point Method for large-deformation geomechanics | HPC | Open Source |
| [Ansys](https://www.ansys.com/) | Structural analysis and CFD via Ansys Workbench | HPC (TAP) | Licensed |
| [EE-UQ](https://simcenter.designsafe-ci.org/research-tools/ee-uq/) | Earthquake engineering with uncertainty quantification (SimCenter) | HPC | Open Source |
| [Hydro-UQ](https://simcenter.designsafe-ci.org/research-tools/hydro-uq/) | Water loading assessment for tsunami and storm surge (SimCenter) | HPC | Open Source |
| [WE-UQ](https://simcenter.designsafe-ci.org/research-tools/we-uq/) | Wind engineering with uncertainty quantification (SimCenter) | HPC | Open Source |
| [PBE](https://simcenter.designsafe-ci.org/research-tools/pbe-application/) | Performance-based engineering from multiple hazards (SimCenter) | HPC | Open Source |
| [quoFEM](https://simcenter.designsafe-ci.org/research-tools/quofem-application/) | Uncertainty quantification and optimization with FE (SimCenter) | HPC | Open Source |
| [R2D](https://simcenter.designsafe-ci.org/research-tools/r2d/) | Regional resilience and impact assessment (SimCenter) | HPC | Open Source |
| [Dakota](https://dakota.sandia.gov/) | Optimization and uncertainty quantification | JupyterHub | Open Source |
| [Clawpack](https://www.clawpack.org/) | Conservation laws for hyperbolic systems (wave propagation) | JupyterHub | Open Source |

## Analysis

| Application | Description | Execution | License |
|---|---|---|---|
| [Jupyter](https://jupyter.designsafe-ci.org/) | Notebooks with Python, R, and access to TACC HPC via dapi | JupyterHub | Open Source |
| [MATLAB](https://www.mathworks.com/products/matlab.html) | Data analysis, algorithms, and modeling | VM | Licensed |
| HVSRweb | Horizontal-to-vertical spectral ratio for seismic site analysis | Web App | Open Source |
| SWbatch | Batch surface wave inversion using Geopsy/Dinver | Web App | Open Source |

## Visualization

| Application | Description | License |
|---|---|---|
| [ParaView](https://www.paraview.org/) | Data analysis and visualization for large datasets | Open Source |
| [VisIt](https://visit-dav.github.io/visit-website/) | Configurable visualization for large datasets | Open Source |
| [STKO](https://asdeasoft.net/stko/) | Scientific ToolKit for OpenSees visualization | Open Source |
| [GiD](https://www.gidsimulation.com/) | Mesh geometry and pre/post-processing | Licensed |
| Kalpana | Convert ADCIRC output to GIS shapefiles | Open Source |
| FigureGen | Image generation from ADCIRC output files | Open Source |
| [Potree](https://potree.github.io/) | Point cloud visualization for LiDAR data | Open Source |

## GIS and Field Data

| Application | Description | License |
|---|---|---|
| Hazmapper | Geospatial data visualization with geotagged photos and GPS tracks | Open Source |
| [QGIS](https://qgis.org/) | Geographic information system for geospatial analysis | Open Source |
| Taggit | Image organization, grouping, and tagging | Open Source |

## Hazard Data

| Application | Description | License |
|---|---|---|
| Ground-Motion Portal | Generate seismograms for historical and scenario earthquakes | Open Source |
| NGL (Next-Generation Liquefaction) | Liquefaction database access with CPT Viewer | Open Source |
| TPU Wind | Wind pressure databases from wind tunnel tests | Open Source |
| VORTEX-Winds DEDM-HR | Wind tunnel measurement databases | Open Source |

---

## Tapis App IDs

When submitting jobs through [dapi](https://designsafe-ci.github.io/dapi/), the `app_id` parameter specifies which application to run. The table below lists common app IDs. Use `ds.apps.list()` for the full current list.

| Application | `app_id` | System | MPI | Notes |
|---|---|---|---|---|
| OpenSees MP | `opensees-mp-s3` | Stampede3 | Yes | Parallel structural analysis |
| OpenSees SP | `opensees-sp-s3` | Stampede3 | Yes | Single-process parallel |
| OpenSees EXPRESS | `opensees-express` | VM | No | Serial Tcl on shared VM |
| OpenFOAM | `openfoam-s3` | Stampede3 | Yes | CFD simulations |
| ADCIRC | `adcirc-s3` | Stampede3 | Yes | Storm surge modeling |
| LS-DYNA | `ls-dyna-s3` | Stampede3 | Yes | Explicit dynamics |
| MPM | `mpm-s3` | Stampede3 | Yes | Material point method |
| Agnostic App | `designsafe-agnostic-app` | Stampede3 | Configurable | General-purpose (Python, Tcl, PyLauncher) |
| OpenSeesPy | `openseespy-s3` | Stampede3 | No | Python-based OpenSees |

App IDs may include version suffixes (e.g., `opensees-mp-s3-3.7.0`). When no version is specified, the latest version is used.

---

The full list of applications is maintained at [DesignSafe Tools & Applications](https://www.designsafe-ci.org/use-designsafe/tools-applications/). The [SimCenter](https://simcenter.designsafe-ci.org/) provides additional research and learning tools beyond those listed here.

Detailed configuration guides exist for [OpenSees](opensees.md), [OpenFOAM](openfoam.md), [ADCIRC](adcirc.md), and the [Agnostic App](agnostic-app.md).
