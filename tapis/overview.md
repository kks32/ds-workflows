# Tapis Overview

In the previous chapters, you learned how jobs run on TACC systems, including how SLURM schedules work, how resources are requested, how queues behave, and what happens on compute nodes once a job starts. Tapis is the layer that automates all of that complexity.

Tapis is a network-based platform that provides a consistent and secure way to submit, monitor, and manage computational jobs and data across high-performance computing (HPC) and cloud systems. Rather than writing and managing scheduler scripts, staging files by hand, and tracking outputs manually, you describe *what* you want to run, and Tapis handles *how* and *where* it runs.

In practical terms, Tapis sits between your working environment (for example, Jupyter notebooks, Python scripts, or the DesignSafe portal) and the execution systems at Texas Advanced Computing Center (TACC). It translates high-level job requests into the low-level scheduler actions you already know about, then tracks the full lifecycle of each job.

<img src="../images/ComputeWorkflow.jpg" alt="Compute Environment" width="75%" />

---

## Tapis Jobs

A Tapis Job represents one execution of a registered Tapis App, including

* Input files and parameters
* Resource requests (nodes, cores, walltime, queues)
* Execution and archiving instructions

When you submit a job through Tapis, you are effectively saying

> "Run this application with these inputs and resource requirements on that system."

Behind the scenes, Tapis

* Stages your input data on the execution system
* Generates and submits the appropriate scheduler job
* Monitors job state and execution progress
* Collects logs, exit codes, and output files
* Archives results and metadata for later use

From the user's perspective, this replaces dozens of manual steps with a single, repeatable job submission.

---

## How Tapis Fits into the DesignSafe Workflow

Within DesignSafe, Tapis connects your analysis to TACC's compute resources. Whether you submit a job from the web portal or from a Python notebook using Tapipy, the same Tapis services handle execution, monitoring, and data management.

This separation of concerns is intentional.

* You focus on your analysis
* Tapis handles execution logistics
* TACC systems provide the compute power

---

## Why Use Tapis?

Tapis provides

* Automation of SLURM-based workflows, without requiring you to manage job scripts directly
* Seamless access to powerful HPC systems at TACC through DesignSafe
* End-to-end tracking of inputs, parameters, execution details, and outputs
* Reproducibility, by preserving job metadata and app versions
* Collaboration and sharing, with controlled access to apps, jobs, and data
* No infrastructure to maintain, because Tapis is hosted and operated for you

---

## What Can Tapis Do?

Beyond job submission, Tapis supports

* File management on Tapis-enabled systems
* Job monitoring, including status, logs, and exit codes
* Job metadata queries for provenance and reproducibility
* App-based execution, enabling standardized and versioned analyses

All of these capabilities are accessible programmatically. In this document we focus primarily on Python-based workflows using Tapipy, building on the computational environments you have already seen.

---

## The Tapis Workflow at a Glance

Most DesignSafe applications powered by Tapis follow this pattern.

1. Authenticate with Tapis
2. Stage input files (if needed)
3. Submit a job using a registered Tapis App
4. Monitor job status (pending, running, finished)
5. Access archived outputs for post-processing or downstream analyses

---

## Tapis Online Documentation

1. [Tapis Project Page](https://tapis-project.org/)
2. [Tapis Documentation](https://tapis.readthedocs.io/en/latest/index.html)
3. [Tapipy Documentation](https://tapis.readthedocs.io/en/latest/technical/pythondev.html)
4. [Tapipy GitHub (OpenAPI specifications)](https://github.com/tapis-project/tapipy/blob/main/tapipy/resources/openapi_v3-apps.yml)


# Interfacing with Tapis

Tapis is a platform designed to support remote job execution, data transfer, and workflow automation across heterogeneous systems like supercomputers, cloud nodes, and containers.

The Tapis Jobs API allows users to

* Submit computational jobs based on registered applications
* Monitor job status through a consistent lifecycle
* Retrieve outputs and logs after completion
* Automate workflows with job metadata, permissions, and archiving

Tapis Jobs are tightly integrated with Tapis Systems (where the job runs) and Tapis Apps (which define how it runs).

There are two main ways to interact with Tapis.

::::{dropdown} 1. Tapis SDK (Tapipy)

Tapipy is a Python SDK generated directly from the Tapis OpenAPI specification. It provides

* Full coverage of the Tapis API
* Complete access to all Tapis endpoints and resources
* Methods that closely mirror the REST API structure
* Fine-grained control for advanced workflows

With Tapipy, you do not need to manually construct HTTP requests, handle tokens, or parse raw JSON responses. The SDK manages these details for you.

An SDK (Software Development Kit) is a collection of tools, libraries, documentation, and code examples that help developers build software applications that interact with a specific platform, service, or system. In simple terms, an SDK is a developer toolbox that makes it easier to program against an API or platform.

* Wraps the full REST API into Python methods
* Allows you to focus on your workflow logic, not protocol details
* Tapipy is the official Python SDK for Tapis v3, auto-generated from the Tapis OpenAPI spec (always current)
* Handles authentication, tokens, headers, HTTP requests, and response parsing into Python objects


:::{dropdown} Example (using Tapipy SDK)

```python
    from tapipy.tapis import Tapis

    client = Tapis(base_url="https://tacc.tapis.io",
                   username="your-username",
                   password="your-password",
                   account_type="tacc")

    client.get_tokens()

    # Submit a job
    job_request = {
        "name": "example-job",
        "appId": "hello-world-1.0",
        "archive": True,
        "archiveSystemId": "tacc-archive",
        "archivePath": "your-username/job-output"
    }

    job = client.jobs.submitJob(body=job_request)
    print("Job ID:", job['id'])

    jobs = client.jobs.listJobs()
```

:::


::::

::::{dropdown} 2. Tapis API (REST API)

The Tapis REST API is the raw interface that powers everything behind the scenes.

* Uses standard web protocols (HTTP, HTTPS, GET, POST, PUT, etc.)
* You send requests directly to Tapis endpoints
* Can be accessed using any language (Python, Java, curl, etc.)
* Requires you to build request URLs, add authentication headers, manage tokens, format request bodies (payloads) as JSON, parse JSON responses, and handle HTTP errors and headers manually

:::{dropdown} Example (using curl)

```bash
    curl -X GET https://tacc.tapis.io/v3/jobs \
      -H "Authorization: Bearer <token>"
```
:::

::::


| Approach | Description | Who uses it? |
| ------------------------ | -------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| Tapis API (REST API) | The raw interface exposed by Tapis via HTTP requests (URLs, headers, tokens, JSON payloads). | Advanced users, system integrators, non-Python clients |
| Tapis SDK (Tapipy) | A Python library that wraps the API into easy-to-use Python functions. | Most application developers, researchers, students |


## Why We Recommend Tapipy

The SDK uses the API under the hood. You do not lose power, you just lose the pain.


| Feature | Tapipy |
| ---------------- | ----------------------------------- |
| API coverage | 100% (full Tapis API) |
| Auto-generated | Always current |
| Authentication | Simplified |
| Request building | Pass Python dicts |
| Error handling | Python exceptions |
| Suitable for | Researchers, students, developers |

## SDK and API Compared

* An SDK is a toolkit that simplifies programming against a system like Tapis
* The Tapis API is a low-level interface (web-based, language-agnostic)
* The Tapis SDK is a Python wrapper for the Tapis API (easier for developers)
* SDKs use the API under the hood but wrap APIs with user-friendly methods in a specific programming language

| | Tapis API | Tapis SDK (Tapipy) |
| ------------------------ | ---------- | ------------------ |
| Language | Any | Python |
| Manual token handling | Yes | No |
| Raw HTTP requests | Yes | No |
| Recommended for training | Advanced | Yes |
