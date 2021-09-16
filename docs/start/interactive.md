
# Interactive Sessions

Interactive sessions are a way to use Beaker to quickly experiment
without the overhead of creating an image each time you change your code.

## Select a Machine

Interactive sessions are available on any machine that is running Beaker.
You can see a list of available machines by first selecting a cluster
from the [list of on-premise clusters](https://beaker.org/clusters).
Once you have a chosen cluster, select any node to find its hostname.

## Connect to the Machine

Connect to the machine via SSH. You will need to be on the VPN.
If you have any issues connecting to the machine, contact IT.

## Start a Session

Once on the machine, run `beaker session create` to start a new session.
This will ask for your user token which is used to authenticate with Beaker.
Beaker will then claim resources on the node, which may take a while if the node is full.
Finally, Beaker will create an interactive session for you.

## Interactive Environment

Each interactive session is hosted in a container sandbox.  Within the container
you are free to run or install whatever you want; you won't affect other processes on the machine.

By default, interactive sessions use the `allenai/base:cuda11.2-ubuntu20.04` Docker image which is maintained by the Beaker team.
This image is based on Ubuntu 20.04 and already has CUDA 11.2 drivers installed.
It also comes with some useful tools like the AWS CLI, Google Cloud CLI, and the Beaker CLI.
If you find we've missed a common tool, please contact the team and we will consider adding it.

Your home directory is mounted into the container so anything you write to `~` will persist
between sessions on that node. A few other directories are also available from within sessions.
See [here](../concept/experiments.md#allowedhostpaths) for the full list.

### Reserving GPUs

By default, sessions do not request any resources and won't be assigned a GPU.
If you need GPUs, use the `--gpus <count>` flag when creating a session e.g.:

```
beaker session create --gpus 2
```

Session resources work the same as task resources, which are described [here](../concept/experiments.md#TaskResources)
Node that sessions that don't request GPUs are not guaranteed any memory and may be killed if there is contention for memory on the machine.

### Running a Command

By default, interactive sessions run a Bash shell.
If you would like to run a command instead, you can pass additonal arguments when creating a session
and they will be passed along to the Docker container.

For example, the command below prints the Python version.
You could also use this to run a Python script in your home directory.

```
beaker session create -- python3 --version
```

### Alternative Base Images

If you need to use a different base image, pass the `--image` flag when creating a session.

Beaker Interactive supports both [Docker](https://docker.com) and [Beaker](../concept/images)
images. To use a Docker image, prefix the image name with `docker://`. To use a
Beaker image use the `beaker://` prefix instead.

For example, this command uses the AllenNLP base image and runs the `test-install` command:

```
beaker session create --image docker://allennlp/allennlp -- allennlp test-install
```

Some features, like user mapping, will only work with the official
base image (`allenai/base:cuda11.2-ubuntu20.04`) or images that extend from it. For instance,
In the AllenNLP image, you will see a prompt like `I have no name!@fd82c7800efa:~$`.
Please contact the Beaker team if you need help setting up a custom environment for your
interactive sessions.

### Docker

Sessions have access to Docker for building and pushing images.
The base image has the `docker` command preinstalled.

**Important note on running Docker containers:**
Please don't use `docker run` to start containers in your interactive session.

Since sessions connect to the host's Docker daemon, any containers started through Docker will
run outside of the session and won't have access to the session's resources. This could cause
resource contention and host instability.

Instead of using Docker to run containers, please exit the interactive session and create a
new one or use a Beaker batch job to run the workload.

### Building Custom Images

At some point you might need to install additional software in your interactive session.
By default, the changes that you make inside a session won't persist after the session stops.
To persist those changes you'll need to save the resulting image with the `--save-image`/`-s` flag.
When this flag is enabled, Beaker will wait for the session to exit and then write the contents
of the root filesystem to a new image in the session's workspace. This will save all packages
installed to the root filesystem such as those installed with `apt-get`. Note that any files
written to mounted volumes like `/home` or `/net` will not be saved.

**Important note on secrets:**
Never write secrets to the filesystem when using `--save-image` since they will be
included in the resulting image. Use secret mounts instead.

For this example, we will create an image with `python2` installed.

First, create a session with the `--save-image` flag:

```
beaker session create --save-image
```

Then, install `python2`:

```
sudo apt-get update
sudo apt-get install python2
```

Now, exit the session with `Ctrl-D`.
Beaker will automatically create an image in your default workspace and wait for the
image to be pushed. Once the image is pushed, you can create a new session with it:

```
beaker session create --image beaker://<image-id>
```

...and you should now be able to use `python2` like so:

```
[beaker] name ~ $ python2
Python 2.7.18 (default, Mar  8 2021, 13:02:45)
[GCC 9.3.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

Give the image a name to make it easier to find in the future:

```
beaker image rename <image-id> <new-name>
```

## Secrets

[Secrets](../../concept/secrets.md) can be used in sessions through environment variables or files.

To use a secret as an environment variable:

```
beaker session create --secret-env <secret name>=<environment variable>
```

To mount a secret as a file:

```
beaker session create --secret-mount <secret name>=<file path>
```

## Git Authentication

If you already use an SSH key to authenticate with GitHub, you can easily add
it to Beaker and use it in interactive sessions. First, add the SSH key to Beaker:

```
cat ~/.ssh/id_rsa | beaker secret write <workspace> ssh-key
```

Then, mount the SSH key into a Beaker session:

```
beaker session create --secret-mount ssh-key=~/.ssh/id_rsa
```

Test that the SSH key is working:

```
ssh -T git@github.com
```

To delete your SSH key from Beaker:

```
beaker secret delete <workspace> ssh-key
```

### Git Configuration

If you need to make a commit from a session, you will need to configure Git
with your name and email. You can store Git configuration in a secret an mount it into a session, just like an SSH key. On your machine, run:

```
cat ~/.ssh/.gitconfig | beaker secret write <workspace> git-config
```

Then create a session with your Git config:

```
beaker session create --secret-mount ssh-key=~/.ssh/id_rsa --secret-mount git-config=~/.gitconfig
```


## Reattaching to a Session

If you lose connection to a session and want to reattach to it, use `beaker session attach`.

First, find the ID of the session you want to attach to:

```
beaker session list
```

Then, reattach to the session:

```
beaker session attach <session>
```

## Forking a Session

Sessions can run multiple commands at once.
This can be useful if you need to check on the main process of a session without interrupting it.

To get a new terminal in an existing session, run:

```
beaker session exec <session>
```

When you quit this terminal, the session will continue to run.
The session will stop when the main process exits.

You can also provide a command to `exec`. This example prints GPU utilization:

```
beaker session exec <session> -- nvidia-smi
```

## Stopping a Session

To stop a session that you are not attached to, run:

```
beaker session stop <session>
```
