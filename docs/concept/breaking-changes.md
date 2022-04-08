# Breaking Changes in Beaker

On Wednesday, April 20 2022, Beaker will require that all experiment specs use the `v2` format. The **`v1` and `v2-alpha` specifications will be ***deprecated***.

## How will this change affect my experiment?

If your spec is using the `v1` or `v2-alpha` versions, you will not be able to create experiments.

## How do I upgrade and avoid any downtime?

Easy!

- If you're using `v2-alpha` already, it will be as simple as modifying the version in your spec to `v2`. 

- If you're using `v1` there will be a couple steps involved to make your spec compatible. Below is the format of a `v2` spec with all the possible bells and whistles.

```yaml
version: v2
description: this is an example v2 spec
tasks:
- name: task1
  image:
    # pick exactly one of:
    beaker: name or ID of your beaker image
    docker: Reference (SHA or name) of a local or remote Docker image, including registry.
  command: [ any commands you may need the shell to run for this experiment]
  arguments: [ arguments that you need to specify to go along with commands ]
  envvars: 
    name: env var name
    value: optional source the environment variable from a literal value
    secret: optional source the environment variable from a secret in the experiment's workspace.
  datasets:
    mountPath:  absolute path within the container to mount the data source
    subPath: optional sub path to a file or directory within a mounted data source
    source:
      # pick exactly one of:
      beaker: beaker data set by name or ID
      hostPath: source data from your host path
      result: source data from a previous task by name
      url: source data from a cloud service provider
      secret: source data from a sercret in the same workspace as the experiment
  result:
    path: path to where the task will write result output
  resources:
    cpuCount: ex - 4, .5
    gpuCount: ex - 1, 5
    memory: ex - 2.5 GiB
    sharedMemory: ex - 2.5 GiB (default is 5 GiB)
  context:
    cluster: required name or ID of a cluster on which the task should run
    priority: urgent, high, normal (normal is the default), low 
```

You may notice that the authorToken, dependsOn, and minimum memory requirement fields have been deprecated.

@sam or michal, what should I say about why we're deprecating authorToken and dependsOn??

Minimum memory requirement fields are being deprecated because memory is no longer preallocated. You may set a maximum limit for the amount of memory your experiments will use.

## Questions?

Feel free to reach out to us on the [Beaker Users Slack Channel](https://allenai.slack.com/archives/C6MN19S05)
