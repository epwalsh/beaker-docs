# Experiments and Tasks

## Experiments

An experiment is the main unit of execution in Beaker.  For example, an experiment could be:

* Running wordcount over Moby Dick.
* Training the BiDAF model within AllenNLP.
* Parsing millions of sentences from the [Gigaword Corpus](https://catalog.ldc.upenn.edu/LDC2003T05).

Experiments have their own page and URL, allowing the results to be shared across the company or the internet.

![Experiment example](../images/beaker_ex.png)

## Tasks

Tasks are Beaker's fundamental unit of work.  A Beaker experiment may contain multiple tasks.  A task may also depend on the results of another task in its experiment, creating an execution graph.

## Spec Format

Below can be found a detailed description of Beaker's experiment specification format. Specs must be
written in YAML or JSON, as in the following example.

```yaml
version: v2-alpha
description: A simple 'hello world' example.
tasks:
- name: echo
  image:
    docker: busybox:latest
  command: ['/bin/sh', '-c']
  arguments: ['echo Hello world!']
  result: # Required even if the task produces no files.
    path: '/unused'
```

### Experiment

Experiment is the root level structure, describing a single task or an ordered
pipeline of tasks to run.

| Field | Type | Required? | Description |
| ----- | ---- | :-------: | ----------- |
| version | string | Yes | Must be `v2-alpha` |
| description | string | No | Long-form explanation for an experiment |
| tasks | [Task](#task) \[\] | Yes | Specifications for each process to run |

### Task

Task describes a single job, or process.

| Field | Type | Required? | Description |
| ----- | ---- | :-------: | ----------- |
| name | string | No\* | Name is used for display and to refer to the task throughout the spec. It must be unique among all tasks within its experiment. |
| image | [ImageSource](#imagesource) | Yes | A base image to run, usually built with Docker |
| command | string \[\] | No | Command is the full shell command to run as a list of separate arguments. If omitted, the image's default command is used, for example Docker's `ENTRYPOINT` directive. If set, default commands such as Docker's `ENTRYPOINT` and `CMD` directives are ignored.<br><br>Example: `["python", "-u", "main.py"]` |
| arguments | string \[\] | No | Arguments are appended to the `command` and replace default arguments such as Docker's `CMD` directive. If `command` is omitted, arguments are appended to the default command, Docker's `ENTRYPOINT` directive.<br><br> Example: If Command is `["python", "-u", "main.py"]`, specifying arguments `["--quiet", "some-arg"]` will run the command `python -u main.py --quiet some-arg`. |
| envVars | [EnvVar](#envvar) \[\] | No | List of environment variables passed to the container |
| datasets | [DataMount](#datamount) \[\] | No | External data sources mounted into the task as files |
| result | [ResultSpec](#resultspec) | Yes | Where the task will place output files |
| resources | [TaskResources](#taskresources) | No | External hardware requirements, such as memory or GPU devices |
| context | [Context](#context) | Yes | Context describes how and where this task should run. |

\* Though tasks may be anonymous for compatibility reasons, we recommend always
assigning a name.

### ImageSource

ImageSource describes where Beaker can find a task's image. Beaker will automatically pull, or
download, this image immediately before running the task.

| Field | Type | Required? | Description |
| ----- | ---- | :-------: | ----------- |
| beaker | string | No\* | Name or ID of a [Beaker image](./images.md) |
| docker | string | No\* | Tag of a Docker image hosted on the [Docker Hub](https://hub.docker.com) or a private registry. If the tag is from a private registry, the cluster on which the task will run must be pre-configured to enable access. |

\* Exactly one field must be defined.

### EnvVar

An EnvVar defines an environment variable within a task's container.

| Field | Type | Required? | Description |
| ----- | ---- | :-------: | ----------- |
| name | string | Yes | Name of the environment variable following Unix rules. Environment variable names are case sensitive and must be unique. |
| value | string | No | Literal value which can include spaces and special characters. Defaults to "" if no other source is set. |
| secret | string | No | Source the enviroment variable from a [secret](./secrets.md) in the experiment's workspace. |

### DataMount

DataMount describes how to mount a dataset into a task. All datasets are mounted read-only.

| Field | Type | Required? | Description |
| ----- | ---- | :-------: | ----------- |
| mountPath | string | Yes | The mount path is where Beaker will place the dataset within the task container. Mount paths must be absolute and may not overlap with other mounts.<br><br>Because some environments use case-insensitive file systems, mount paths differing only in capitalization are disallowed. |
| subPath | string | No | Sub-path to a file or directory within the mounted dataset. Sub-paths may be used to mount only a portion of a dataset; files outside of the mounted path are not downloaded.<br><br>Example: Given a dataset containing a file `/path/to/file.csv`, setting the sub-path to `path/to` will result in the task seeing `{mountPath}/file.csv`. |
| source | [DataSource](#datasource) | Yes | Location from which Beaker will download the dataset |

### DataSource

| Field | Type | Required? | Description |
| ----- | ---- | :-------: | ----------- |
| beaker | string | No\* | Name or ID of a [Beaker dataset](./datasets.md). Beaker datasets provide the best download performance and are preferred for frequently used datasets. |
| hostPath | string | No\* | Path to a file or directory on the host. The executing host must be configured to allow access to this path or one of its parent directories. See below for a list of available host paths. |
| result | string | No\* | Name of a previous task whose result will be mounted. A result source implies a dependency, meaning this task will not run until its parent completes successfully. |
| url | string | No\* | URL is a web location from which to download data. Beaker currently supports S3 (`s3://`), GCS (`gs://`), and HTTP(S) (`https://`) URLs.
| secret | string | No\* | Name of a [secret](./secrets.md) within the experiment's workspace which will be mounted as a plain-text file. |

\* Exactly one source field must be defined.

#### Allowed Host Paths

The following host paths are allowed on every on-premise machine managed by the Beaker team:
 - `/net` for access to NFS.
 - `/raid` for access to RAID.
 - `/var/beaker/share` as a shared local scratch space. All Beaker workloads have full access to files in this path, so don't put sensitive information there. Also to avoid collisions it's probably worth generating a subpath that's unlikely to be used by another process.

### ResultSpec

ResultSpec describes how to capture a task's results. Results are captured as datasets from the
given location. Beaker monitors this location for changes and periodically uploads files as they
change in near-real-time.

| Field | Type | Required? | Description |
| ----- | ---- | :-------: | ----------- |
| path | string | Yes | Directory to which the task will write output files. |

### Context

A Context describes an execution environment, or how a task should be run. Because contexts depend
on external configuration, a given context may be invalid or unavailable if a task is re-run at a
future date.

| Field | Type | Required? | Description |
| ----- | ---- | :-------: | ----------- |
| cluster | string | Yes | Name or ID of a Beaker cluster on which the task should run. |
| priority | string | No | Set priority to change the urgency with which a task will run.<br><br>Values may be `low`, `normal`, or `high`. Tasks with higher priority are placed ahead of tasks with lower priority in the queue.<br><br>Priority may also be elevated to `urgent` through UI. |

### TaskResources

TaskResources describe minimum external hardware requirements which must be available for a task to
run.

TaskResources are optional. If a task does not specify any resource requirements, it will:
 - Run immediately.
 - Not have access to GPUs.
 - Not have a memory limit.
 - Be killed if the host machine is running out of memory.

If a task specifies resource requirements, it will claim a slice of the machine proportional to its largest resource request.
Each machine is divided into a number of slots equal to the number of GPUs it has, or 1 slot if it has no GPUs.
Tasks that specify resource requirements are allocated enough slots to accomodate their requirements.

For example, if a machine has 8 GPUs and 80GiB of memory, it will be divided into 8 slots, each with 1 GPU and 10GiB of memory.
If a job requests 1 GPU, it will get 1 GPU and 10GiB of memory.
If it requests 15 GiB of memory, it will get 2 GPUs and 20GiB of memory.
Note that in this example, the job is allocated 2 GPUs even though it didn't request any.

If a task specifies resource requirements, it will:
 - Wait to run until there are sufficient resources available on the machine.
 - Have guaranteed access to at least the resources it requested.
 - Have limited memory proportional to the number of slots it is using.
 - Not be killed if the host machine is running out of memory.

Note on CPUs: The amount of CPU a job can use is only limited when there is contention.
During periods of contention, each job gets a share of the CPU proportional to the number of slots it is using.

| Field | Type | Required? | Description |
| ----- | ---- | :-------: | ----------- |
| cpuCount | double | No | Minimum number of logical CPU cores. It may be fractional. Since CPU is only limited during periods of contention, it's generally not necessary to specify this field.<br><br>Examples: `4`, `0.5`. |
| gpuCount | int | No | Minimum number of GPUs. It must be non-negative. |
| memory | string | No | Minimum available system memory as a number with unit suffix. <br><br>Examples: `2.5GiB`, `1024m`. |
| sharedMemory | string | No | Size of /dev/shm as a number with unit suffix. <br><br>Examples: `2.5GiB`, `1024m`. |

## Older Spec Formats

### V1

The following example outlines most features of the current experiment spec.

```yaml
version: v1
tasks:
- name: my-first-task

  # (required) Cluster sets where the task should run.
  # See 'beaker cluster create'
  cluster: ai2/my-cluster

  # Spec describes the specifics of a process to run.
  spec:
    # (optional) The Beaker image name or ID. Either this or dockerImage is required.
    image: myaccount/myimage

    # (optional) The DockerHub image tag. Either this or image is required.
    dockerImage: busybox:latest

    # (required) Where results will be placed.
    # Beaker automatically uploads files as the job runs.
    resultPath: /path/to/results

    # (optional) Override the default command in the image.
    args: ['python', '-u', '/code/run.py']

    # (optional) Environment variables as a key/value map.
    env:
      A_NUMBER: '0.5'
      A_STRING: foobar
      # ...

    # (optional) Mount datasets as directories into the task.
    datasetMounts:
      - datasetId: training-files
        containerPath: /traindata

      # Mount '/foo.yaml' from a config dataset to '/path/to/config.yml'.
      # The subPath field mounts files or subdirectories within a dataset.
      - datasetId: config-dataset
        subPath: foo.yaml
        containerPath: /path/to/config.yml

    # (optional) Set minimum resource requirements for larger tasks.
    # All fields are optional, and any number can be set.
    requirements:
      memory: '7.5GiB' # number followed by unit suffix
      cpuCount: 1.5    # number of logical cores
      gpuCount: 2      # must be an integer

- name: my-second-task
  spec:
    # ...

  # (optional) Set dependencies between tasks.
  dependsOn:
      # (required) Name the task to depend on.
    - parentName: my-first-task

      # (optional) If set, mount the result of that task to the following directory.
      containerPath: /path/to/input
```
