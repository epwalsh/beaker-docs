# Your First Experiment

By now you are probably eager to run an experiment with your code in Beaker.  This tutorial will take you through the steps required to run your experiment.

This tutorial assumes that your experiment needs GPUs, is built with PyTorch, and consists of a single command.  If these assumptions don't hold for your experiments, then these general steps will still apply but you might need to make some more modifications to actually run your experiment.

This example assumes you've successfully [installed Beaker and Docker](install.md). This particular example also uses GPUs and so requires installation of the [NVidia container toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html). Note: The NVidia container toolkit is strictly for Linux. The toolkit is already installed in AI2's on-prem machines, but if you're working with Beaker locally on another OS, you will encounter an assertion failure in main.py (in the example code from https://github.com/beaker/pytorch-example).

## Creating a Docker Image for Your Code

As a first step, you need to create a Docker image for your code so Beaker can run it on a cluster.  Fortunately, for simple Python applications this is quite straightforward.  The first step is to create a `Dockerfile` in the same directory as your code.  Below is a sample Dockerfile that can be adapted to your project.

```Dockerfile
# The base image, which will be the starting point for the Docker image.
# We're using a PyTorch image built from https://github.com/allenai/docker-images
# because PyTorch is really big we want to install it first for caching.
FROM pytorch/pytorch:1.9.0-cuda11.1-cudnn8-runtime

ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8

# This is the directory that files will be copied into.
# It's also the directory that you'll start in if you connect to the image.
WORKDIR /stage/

# Copy the `requirements.txt` to `/stage/requirements.txt/` and then install them.
# We do this first because it's slow and each of these commands are cached in sequence.
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy the file `main.py` to `/stage/main.py`
# You might need multiple of these statements to copy all the files you need for your experiment.
COPY main.py .

# Copy the folder `scripts` to `scripts/`
# You might need multiple of these statements to copy all the folders you need for your experiment.
COPY scripts scripts/
```

Copy the above Dockerfile to the same directory as your code, and modify it so it copies over all the files needed to run your experiment.  As an example, in the rest of this tutorial I will use the code in https://github.com/beaker/pytorch-example.

Now that you have a Dockerfile, you can build it with `docker build -t my-experiment .`.  This will take a while the first time because building the Docker image involves downloading and installing PyTorch, but the second time you build this will be cached and be much faster.  Once your image is built, you can try it out locally with `docker run -it --rm --gpus 0 my-experiment python main.py`.

The `my-experiment` name is the image tag. You can use any string you'd like--we just use `my-experiment` here as an example.


## Creating a Beaker Specification for Your Experiment

Now that we've tried out running the Docker image locally, and we're relatively sure things work, it's time to upload it to Beaker.  Since PyTorch is pretty big, it's best to do this from a location with fast upload speeds.

```
beaker image create --name my-experiment my-experiment
```

Now that we have a Docker image we can begin to define our experiment.  We will do this by writing a YAML file called `beaker-config.yaml` that tells Beaker information like what image to use and where to run the experiment.

```yaml
version: v2-alpha
description: My Experiment
tasks:
  # We only have one step in our experiment, so there's only one entry in this list
  - name: training
    image:
      # You will want to replace `username` below with your Beaker username
      beaker: username/my-experiment
    command: [python, -u, main.py]
    result:
      # Beaker will capture anything that's written to this location and store it in the results
      # dataset.
      path: /output
    resources:
      gpuCount: 1
    context:
      cluster: ai2/general-cirrascale
      priority: normal
```

For a more compilicated experiment you may need more configuration, perhaps to link together multiple tasks or to mount in a dataset that your experiment needs.  For more details, please see [the experiment spec](https://github.com/beaker/docs/blob/main/docs/concept/experiments.md#spec-format).

Now that we have our Docker image uploaded and our experiment spec written, we can run our job on Beaker with a single command.

```
beaker experiment create beaker-conf.yaml
```

The command will output a beaker URL that you can visit to see details about your experiment.

That's it!  While that took a bit of setup, now moving our job to another cluster is simply a one line change, and it would be easy to script a large hyperparameter tuning.  Also, our results are persisted forever on Beaker.org, so if we ever need to check back on our experiment we can just look there.

