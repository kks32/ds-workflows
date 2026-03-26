# DesignSafe Workflows

[![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![DesignSafe](https://img.shields.io/badge/DesignSafe-CI-blue)](https://designsafe-ci.org)
[![Authors](https://img.shields.io/badge/Authors-DesignSafe--CI-orange)](AUTHORS.md)

A guide to running computational workflows on [DesignSafe](https://designsafe-ci.org), from interactive exploration in a Jupyter notebook to production-scale simulations on [TACC](https://www.tacc.utexas.edu/) supercomputers.

## What this guide covers

- **[How It Works](guide/how-it-works.md)** The DesignSafe portal, three compute environments (JupyterHub, Virtual Machines, HPC), storage, and how to design a workflow around your research
- **[Running HPC Jobs](guide/job-resources.md)** How jobs flow from submission to results, resource parameters, node types, queues, allocations, and file staging
- **[Debugging Failed Jobs](guide/debugging.md)** Job states, reading output files, troubleshooting checklist, and common failure patterns
- **[Parallel Computing](guide/parallel-computing.md)** MPI concepts, ranks, and when parallel computing is (and is not) needed
- **[Parameter Sweeps](guide/parameter-sweeps.md)** Running hundreds of independent simulations with PyLauncher
- **[DesignSafe Applications](apps/overview.md)** Catalog of 45+ tools for simulation, analysis, visualization, GIS, and hazard data
- **[Advanced Topics](advanced/tapis.md)** Tapis internals, execution strategies, custom app development

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
