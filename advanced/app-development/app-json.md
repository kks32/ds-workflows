# app.json Reference

The `app.json` file is the definition of a Tapis application. It tells Tapis everything about the app: its name, version, execution system, inputs, parameters, and resource defaults. For an overview of how app.json fits into the broader Tapis app model, see [Tapis and Custom Apps](../tapis.md).

External references:

* [Tapis Apps Documentation](https://tapis.readthedocs.io/en/latest/technical/apps.html)
* [Tapis Jobs Documentation](https://tapis.readthedocs.io/en/latest/technical/jobs.html)

## Sample Schema (opensees-mp-s3)

This is the full app.json for the public OpenSeesMP app on Stampede3. It illustrates how inputs, parameters, scheduler profiles, and archive settings are structured in a production app.

```
  sharedAppCtx: "wma_prtl"
  isPublic: True
  tenant: "designsafe"
  id: "opensees-mp-s3"
  version: "latest"
  description: "Runs all the processors in parallel. Requires understanding of parallel processing and the capabilities to write parallel scripts."
  owner: "wma_prtl"
  enabled: True
  versionEnabled: True
  locked: False
  runtime: "ZIP"
  runtimeVersion: None
  runtimeOptions: None
  containerImage: "tapis://cloud.data/corral/tacc/aci/CEP/applications/v3/opensees/latest/OpenSees/opensees.zip"
  jobType: "BATCH"
  maxJobs: 2147483647
  maxJobsPerUser: 2147483647
  strictFileInputs: True
  uuid: "1410a584-0c5e-4e47-b3b0-3a7bea0e1187"
  deleted: False
  created: "2025-02-20T18:01:49.005183Z"
  updated: "2025-07-25T19:55:59.261049Z"
  sharedWithUsers: []
  tags: ["portalName: DesignSafe", "portalName: CEP"]
  jobAttributes: {
    description: None
    dynamicExecSystem: False
    execSystemConstraints: None
    execSystemId: "stampede3"
    execSystemExecDir: "${JobWorkingDir}"
    execSystemInputDir: "${JobWorkingDir}"
    execSystemOutputDir: "${JobWorkingDir}"
    dtnSystemInputDir: "!tapis_not_set"
    dtnSystemOutputDir: "!tapis_not_set"
    execSystemLogicalQueue: "skx"
    archiveSystemId: "stampede3"
    archiveSystemDir: "HOST_EVAL($WORK)/tapis-jobs-archive/${JobCreateDate}/${JobName}-${JobUUID}"
    archiveOnAppError: True
    isMpi: False
    mpiCmd: None
    cmdPrefix: None
    nodeCount: 2
    coresPerNode: 48
    memoryMB: 192000
    maxMinutes: 120
    fileInputs: [
      {
        name: "Input Directory"
        description: "Input directory that includes the tcl script as well as any other required files. Example input is in tapis://designsafe.storage.community/app_examples/opensees/OpenSeesMP"
        inputMode: "REQUIRED"
        autoMountLocal: True
        envKey: "inputDirectory"
        sourceUrl: None
        targetPath: "inputDirectory"
        notes: {
          selectionMode: "directory"
        }
      }
    ]
    fileInputArrays: []
    subscriptions: []
    tags: []
    parameterSet: {
      appArgs: [
        {
          arg: "OpenSeesMP"
          name: "mainProgram"
          description: None
          inputMode: "FIXED"
          notes: {
            isHidden: True
          }
        }
        {
          arg: None
          name: "Main Script"
          description: "The filename only of the OpenSees TCL script to execute. This file should reside in the Input Directory specified. To use with test input, use 'freeFieldEffective.tcl'"
          inputMode: "REQUIRED"
          notes: {
            inputType: "fileInput"
          }
        }
      ]
      containerArgs: []
      schedulerOptions: [
        {
          arg: "--tapis-profile OpenSees_default"
          name: "OpenSees TACC Scheduler Profile"
          description: "Scheduler profile for the default version of OpenSees"
          inputMode: "FIXED"
          notes: {
            isHidden: True
          }
        }
      ]
      envVariables: []
      archiveFilter: {
        includeLaunchFiles: True
        includes: []
        excludes: []
      }
      logConfig: {
        stdoutFilename: ""
        stderrFilename: ""
      }
    }
  }
  notes: {
    icon: "OpenSees"
    label: "OpenSeesMP"
    helpUrl: "https://www.designsafe-ci.org/user-guide/tools/simulation/#opensees-user-guide"
    category: "Simulation"
    isInteractive: False
    hideNodeCountAndCoresPerNode: False
  }

```

## Application Attributes

| Attribute | Type | Example | Notes |
| ---------------- | ------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------ |
| tenant | String | designsafe | Name of the tenant for which the application is defined. `tenant + version + id` must be unique. |
| id | String | my-ds-app | Name of the application. URI safe. `tenant + version + id` must be unique. Allowed characters: alphanumeric `[0-9a-zA-Z]`, `[-.\_\~]`. Required at creation. |
| version | String | 0.0.1 | Version of the application. URI safe. `tenant + version + id` must be unique. Allowed characters: alphanumeric `[0-9a-zA-Z]`, `[-.\_\~]`. Required at creation. |
| description | String | A sample application | Optional description |
| owner | String | jdoe | Username of the owner. Default is `${apiUserId}` |
| enabled | boolean | FALSE | Whether app is enabled. Default is TRUE. |
| versionEnabled | boolean | FALSE | Whether version is enabled. Default is TRUE. |
| locked | boolean | FALSE | Indicates if version is currently locked. Locking disallows updates. Default is FALSE. |
| runtime | enum | SINGULARITY | Runtime to be used when executing the application. Options are DOCKER, SINGULARITY, ZIP. Default is DOCKER. |
| runtimeVersion | String | 2.5.2 | Optional version or range of versions. |
| runtimeOptions | \[enum] | | Options that apply to specific runtimes. Options are NONE, SINGULARITY\_RUN. Default is NONE. WARNING: SINGULARITY_START is no longer supported. |
| containerImage | String | docker.io/hello-world | Reference for the container image. Required at creation. |
| jobType | enum | BATCH | Default job type (BATCH, FORK). May be overridden in the job submit request. Default is FORK. |
| maxJobs | int | 10 | Max number of jobs that can be running for this app on a system. Set to -1 for unlimited. Default is unlimited. |
| maxJobsPerUser | int | 2 | Max number of jobs per job owner. Set to -1 for unlimited. Default is unlimited. |
| strictFileInputs | boolean | FALSE | TRUE disallows unnamed file inputs. If TRUE then a job request may only use named file inputs defined in the app. Default is FALSE. |
| jobAttributes | JobAttributes | | Attributes related to job execution. See below. |
| tags | \[String] | | List of tags as simple strings. |
| notes | String | {"project": "myproj"} | Simple metadata as a JSON object. Not used by Tapis. |
| uuid | UUID | 20281 | Auto-generated by service. |
| created | Timestamp | 2020-06-19T15:10:43Z | When the app was created. Maintained by service. |
| updated | Timestamp | 2020-07-04T23:21:22Z | When the app was last updated. Maintained by service. |

## jobAttributes

| Attribute | Type | Example | Notes |
| ---------------------- | ----------------- | -------------------------- | ----------------------------------- |
| description | String | | Description to be filled in when this application is used to run a job. Macros allow this to act as a template to be filled in at job runtime. |
| execSystemId | String | | Specific system on which the application is to be run. |
| execSystemExecDir | String | \${JobWorkingDir}/jobs/... | Application's working directory. Directory where application assets are staged. Current working directory at application launch time. Macro template variables such as ${JobWorkingDir} may be used. Default is ${JobWorkingDir}/jobs/${JobUUID} |
| execSystemInputDir | String | \${JobWorkingDir}/jobs/... | Directory where Tapis stages the inputs required by the application. Default is ${JobWorkingDir}/jobs/${JobUUID} |
| execSystemOutputDir | String | \${JobWorkingDir}/jobs/... | Directory where Tapis expects the application to store its final output results. Files here are candidates for archiving. Default is ${JobWorkingDir}/jobs/${JobUUID}/output |
| dtnSystemInputDir | String | | Directory relative to DTN rootDir to which input files will be transferred. Optional. Default is !tapis_not_set |
| dtnSystemOutputDir | String | | Directory relative to DTN rootDir from which output files will be transferred. Optional. Default is !tapis_not_set |
| execSystemLogicalQueue | String | normal | Queue name |
| archiveSystemId | String | | System to use when archiving outputs. |
| archiveSystemDir | String | \${JobWorkingDir}/jobs/... | Directory on archiveSystemId where outputs will be placed. Default is ${JobWorkingDir}/jobs/${JobUUID} |
| archiveOnAppError | boolean | TRUE | Indicates if outputs should be archived if there is an error while running job. Default is TRUE. |
| isMpi | boolean | FALSE | Indicates that application is to be executed as an MPI job. Default is FALSE. |
| mpiCmd | String | mpirun, ibrun -n 4 | Command used to launch MPI jobs. Prepended to the command used to execute the application. Conflicts with cmdPrefix if isMpi is set. |
| cmdPrefix | String | | String prepended to the application invocation command. Conflicts with mpiCmd if isMpi is set. |
| fileInputs | \[FileInput] | | Collection of file inputs that must be staged for the application. Each input must have a name. strictFileInputs=TRUE means only inputs defined here may be specified for job. |
| fileInputArrays | \[FileInputArray] | | Collection of arrays of inputs that must be staged for the application. Each input must have a name. All inputs in an array have the same target directory. |
| parameterSet | ParameterSet | | Various collections used during job execution. App arguments, container arguments, scheduler options, environment variables. See below. |
| nodeCount | int | | Number of nodes to request during job submission. |
| coresPerNode | int | | Number of cores per node to request during job submission. |
| memoryMB | int | | Memory in megabytes to request during job submission. |
| maxMinutes | int | | Run time to request during job submission. |
| subscriptions | | | Notification subscriptions. |
| tags | \[String] | | List of tags as simple strings. |

## fileInput Attributes

| Attribute | Type | Notes |
| -------------- | ------- | --------------------------------------------- |
| name | String | Identifying label associated with the input. Required at creation time. |
| description | String | Optional description. |
| inputMode | enum | How input is treated when processing job requests. REQUIRED, OPTIONAL, FIXED. Default is OPTIONAL. |
| autoMountLocal | boolean | Indicates if Jobs service should automatically mount file paths into containers. Default is TRUE. |
| sourceUrl | String | Source used by Jobs service when staging file inputs. |
| targetPath | String | Target path used by Jobs service when staging file inputs. |

## fileInputArray Attributes

| Attribute | Type | Notes |
| ----------- | --------- | --------------------------------------------- |
| name | String | Identifying label associated with the input. Required at creation time. |
| description | String | Optional description. |
| inputMode | enum | REQUIRED, OPTIONAL, FIXED. Default is OPTIONAL. |
| sourceUrls | \[String] | Array of sources used by Jobs service when staging file inputs. |
| targetDir | String | Target directory used by Jobs service when staging file inputs. |

## parameterSet Attributes

| Attribute | Type | Notes |
| ---------------- | --------------- | --------------------------------- |
| appArgs | \[Arg] | Command line arguments passed to the application. See Arg attributes below. |
| containerArgs | \[Arg] | Command line arguments passed to the container runtime. |
| schedulerOptions | \[Arg] | Scheduler options passed to the HPC batch scheduler. |
| envVariables | \[KeyValuePair] | Environment variables placed into the runtime environment. Each entry has key (required) and value (optional). See KeyValuePair attributes below. |
| archiveFilter | ArchiveFilter | Sets of files to include or exclude when archiving. Default is to include all files in execSystemOutputDir. |

### appArgs Attributes

| Attribute | Type | Example | Notes |
| ----------- | ------ | -------------------- | ----------------------------------------------------------------- |
| name | String | | Identifying label. Required at creation time. |
| description | String | | Optional description. |
| inputMode | enum | | How argument is treated when processing job requests. Modes: REQUIRED, FIXED, INCLUDE_ON_DEMAND, INCLUDE_BY_DEFAULT. Default is INCLUDE_ON_DEMAND. |
| arg | String | | Value for the argument. Required at creation time. |
| notes | String | {"fieldType": "int"} | Optional metadata as a JSON object. Not used by Tapis. |

### schedulerOptions Attributes

From [Tapis Jobs Documentation](https://tapis.readthedocs.io/en/latest/technical/jobs.html).

Scheduler options are passed to the HPC batch scheduler using that scheduler's conventions. Arguments specified in the application definition are appended to those in the submission request.

Tapis defines a special scheduler option, `--tapis-profile`, to support local scheduler conventions. The Systems service manages SchedulerProfile resources that can be referenced from system definitions. The Jobs service uses directives contained in profiles to tailor application execution to local requirements.

Example profile definition:

```
{
    "name": "TACC",
    "owner": "user1",
    "description": "Test profile for TACC Slurm",
    "moduleLoads": [
        {
            "moduleLoadCommand": "module load",
            "modulesToLoad": ["tacc-singularity"]
        }
    ],
    "hiddenOptions": ["MEM"]
}
```

The `moduleLoads` array contains objects with a `moduleLoadCommand` and a `modulesToLoad` list. The `hiddenOptions` array identifies scheduler options that the local implementation prohibits. Supported options are "MEM" for `--mem` and "PARTITION" for `--partition`.

Jobs will perform macro-substitution on Slurm scheduler options `--job-name` or `-J`, allowing Slurm job names to be dynamically generated before submission.

### envVariables (KeyValuePair) Attributes

Key-value pairs injected as environment variables when the application launches. Pairs from the execution system, application, and job request are aggregated using precedence ordering (system < app < request) to resolve conflicts.

Both key and value are required, though the value can be an empty string.

| Attribute | Type | Example | Notes |
| ----------- | ------ | --------------- | --------------- |
| key | String | INPUT\_FILE | Environment variable name. Required. |
| value | String | /tmp/file.input | Environment variable value |
| description | String | | Optional description |
| inputMode | enum | REQUIRED | Modes: REQUIRED, FIXED, INCLUDE_ON_DEMAND, INCLUDE_BY_DEFAULT. Default is INCLUDE_BY_DEFAULT. |
| notes | String | {} | Simple metadata as a JSON object. Not used by Tapis. |

### archiveFilter Attributes

The archiveFilter conforms to this JSON schema:

```
"archiveFilter": {
   "type": "object",
   "properties": {
      "includes": {"type": "array", "items": {"type": "string", "minLength": 1}, "uniqueItems": true},
      "excludes": {"type": "array", "items": {"type": "string", "minLength": 1}, "uniqueItems": true},
      "includeLaunchFiles": {"type": "boolean"}
   },
   "additionalProperties": false
}
```

An archiveFilter can be specified in the application definition and/or the job submission request. The includes and excludes arrays are merged by appending entries from the application definition to those in the submission request.

The excludes filter is applied first, so it takes precedence over includes. If excludes is empty, no file is explicitly excluded. If includes is empty, all files in execSystemOutputDir will be archived unless explicitly excluded. If includes is not empty, only matching files that are not excluded will be archived.

Each entry is a string, a string with glob wildcards, or a regular expression (prefaced with `REGEX:`). Examples:

```
"myfile.*"
"*2021-*-events.log"
"REGEX:^[\\p{IsAlphabetic}\\p{IsDigit}_\\.\\-]+$"
```

When `includeLaunchFiles` is true (the default), the generated `tapisjob.sh` and `tapisjob.env` files are also archived. These provide useful debugging information but may contain secrets, so handle them accordingly.

| Attribute | Type | Notes |
| ------------------ | --------- | ----------------------------------------------------- |
| includes | \[String] | Files to include when archiving. excludes takes precedence. |
| excludes | \[String] | Files to skip when archiving. excludes takes precedence. |
| includeLaunchFiles | boolean | Whether Tapis-generated launch scripts are included when archiving. Default is TRUE. |

### logConfig

A LogConfig can be supplied in the job submission request and/or in the application definition, with the former overriding the latter. In supported runtimes (currently Singularity and ZIP), the logConfig parameter redirects the application container's stdout and stderr to user-specified files.

### MPI and Related Support

Tapis provides three parameters for launching applications: `mpiCmd`, `isMpi`, and `cmdPrefix`.

The `mpiCmd` parameter sets the MPI launch command. It can be specified in a system definition, application definition, and/or job submission request (lowest to highest priority). For example, if `mpiCmd=mpirun`, the string "mpirun " will be prepended to the command used to execute the application. Some MPI launchers accept their own parameters, such as `mpiCmd=ibrun -n 4`.

The `isMpi` parameter toggles MPI launching on or off. Default is false. When true, `mpiCmd` must be assigned somewhere in the system, application, or job request.

The `cmdPrefix` parameter provides generalized launcher support. Like `mpiCmd`, a cmdPrefix value is prepended to the program's pathname and arguments, but it is not supported in system definitions and does not have a toggle.

`mpiCmd` and `cmdPrefix` are mutually exclusive. If `isMpi` is true, then `cmdPrefix` must not be set.

## inputMode

The inputMode field determines how arguments are handled during job processing.

* **REQUIRED** means the argument must be provided for the job to run. If an arg value is not specified in the application definition, it must be specified in the job request. When provided in both, the job request value overrides the application value.

* **FIXED** means the argument is completely defined in the application and not overridable in a job request.

* **INCLUDE_ON_DEMAND** means the argument will only be included in the final argument list if explicitly referenced in the job request. This is the default.

* **INCLUDE_BY_DEFAULT** means the argument will automatically be included in the final argument list unless explicitly excluded in the job request.

## Scheduler Profiles

A scheduler profile defines the baseline execution environment that TACC provides to your job before your wrapper script runs. It controls how the job is launched, whether the `module` command is available, and which default modules (if any) are preloaded.

Most TACC apps use one of the built-in profiles:

* `tacc-no-modules` loads no modules automatically. Your wrapper script decides exactly which modules to load. This is ideal for custom apps.
* `tacc-apptainer`, `tacc-singularity`, etc. are used for containerized workflows.

For OpenSees/OpenSeesPy apps, the recommended selection is:

```bash
--tapis-profile tacc-no-modules
```

This ensures a clean environment (avoids conflicts between system-loaded and user-requested modules), gives the wrapper script full control, and keeps behavior predictable regardless of user account or portal defaults.

### Scheduler Profiles vs. envVariables

These two mechanisms both influence the job environment but operate at different stages.

Scheduler profiles are executed by TACC before your wrapper script runs. They control how the compute node is initialized, whether the `module` command is available, and whether any default modules are preloaded. A scheduler profile is owned by the execution system, not by your app.

envVariables in app.json are passed directly into your running job and visible within `tapisjob_app.sh`. They allow the app developer or job submitter to express configuration options like which modules to load, which pip packages to install, or whether to use MPI.

| Concern | Scheduler Profile | envVariables |
|---|---|---|
| When it runs | Before your wrapper script | During your wrapper script |
| Who owns it | The execution system | The app developer or job submitter |
| What it controls | System initialization, module availability | Application behavior and configuration |

## Job Submission Attributes vs. App Parameters

When you submit a job to Tapis, the job request includes two kinds of information.

1. General job settings like job name, queue, and resource configuration. These are job-level attributes that tell Tapis how to request resources from the scheduler.

2. Input files and parameter values defined by the app. These are passed into the wrapper script and control how the application behaves.

| Field Type | Used For | Defined In |
| ------------------ | ------------------------------------------------ | --------------------------------------------------- |
| Job attributes | Scheduling, archiving, naming, resource requests | Part of the job request |
| App parameters | Controlling program behavior (flags, values) | Defined in the app template (`app-definition.json`) |

### Processors and Memory

This is where the distinction matters most.

`nodeCount`, `processorsPerNode`, and `memoryPerNode` are job-level attributes. They tell Tapis how many resources to request from the scheduler.

`numProcessors` or `np` (often defined as an app parameter) is a separate value that controls how your script behaves, for example how many MPI processes to launch.

They are related but not the same. You need to coordinate them: ask for enough processors in the job attributes, then tell your wrapper how many processes to use via a parameter. This matters especially when running more than one application within the same Slurm job.

### Example

Job request:

```json
{
  "name": "run-opensees",
  "appId": "openseesmp-3.5.0",
  "nodeCount": 1,
  "processorsPerNode": 4,
  "inputs": {
    "inputScript": "agave://designsafe.storage/path/model.tcl"
  },
  "parameters": {
    "numProcessors": "4"
  }
}
```

Wrapper script:

```bash
ibrun -np ${numProcessors} OpenSeesMP ${inputScript}
```

Here `processorsPerNode` ensures the job requests 4 cores on the system, while `numProcessors` is passed into the app and used in the MPI command.

### Comparison Table

| Feature | Job Submission Attribute | App Parameter |
| ------------------------------ | ------------------------------------------------------------------ | ----------------------------------------------------------- |
| Purpose | Tells Tapis how to run the job on the system | Tells the app how to behave during execution |
| Defined in | Job request (JSON submitted at runtime) | App template (`app-definition.json`) |
| Evaluated by | Tapis system (scheduler, job handler) | Wrapper script (via variable substitution) |
| Appears in wrapper? | Not automatically | Yes, as `${paramId}` |
| Examples | `nodeCount`, `processorsPerNode`, `maxRunTime`, `archive`, `queue` | `numProcessors`, `useDamping`, `solverType`, `writeOutputs` |
| Used for resource control? | Yes (Slurm or batch settings) | Sometimes (e.g., to construct `ibrun -np`) |
| Used for command logic? | No | Yes |

### Job Submission Attributes

These fields are global job settings used by Tapis to manage scheduling, queuing, resource allocation, and archiving.

| Attribute | What it controls | Where it comes from |
| ------------------- | --------------------------------------- | -------------------------------------- |
| `name` | A label for your job | Required in every submission |
| `appId` | The ID of the app you want to run | Matches a registered Tapis App |
| `batchQueue` | Which queue on the cluster to submit to | Optional override (else auto-selected) |
| `nodeCount` | Number of nodes to request | Optional override |
| `processorsPerNode` | Cores per node | Optional override |
| `memoryPerNode` | Memory per node (e.g., `8GB`, `128GB`) | Optional override |
| `maxRunTime` | Walltime (e.g., `02:00:00`) | Optional override |
| `notifications` | URLs to send updates to | Optional |
| `archive` | Whether to archive the results | Optional (default is usually `true`) |
| `archiveSystem` | Storage system to save results to | Optional |
| `archivePath` | Where to place archived outputs | Optional |
