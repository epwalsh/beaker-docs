
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
$ beaker session create --gpus 2 --workspace ai2/your-workspace-name
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
$ beaker session create -- python3 --version
```

### Alternative Base Images

If you need to use a different base image, pass the `--image` flag when creating a session.
You can also update you default image for sessions with `beaker config set default_image <image>`.

Beaker Interactive supports both [Docker](https://docker.com) and [Beaker](../concept/images)
images. To use a Docker image, prefix the image name with `docker://`. To use a
Beaker image use the `beaker://` prefix instead.

For example, this command uses the AllenNLP base image and runs the `test-install` command:

```
$ beaker session create --image docker://allennlp/allennlp -- allennlp test-install
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
$ beaker session create --save-image
```

Then, install `python2`:

```
$ sudo apt-get update
$ sudo apt-get install python2
```

Now, exit the session with `Ctrl-D`.
Beaker will automatically create an image in your default workspace and wait for the
image to be pushed. Once the image is pushed, Beaker will update the default image
in your config file so that future sessions automatically use this image.
If you don't want to update your config, use `--no-update-default-image`
or unset the default image with `beaker config unset default_image`.

```
beaker session create
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
$ beaker image rename <image-id> <new-name>
```

## Secrets

[Secrets](../../concept/secrets.md) can be used in sessions through environment variables or files.

To use a secret as an environment variable:

```
$ beaker session create --secret-env <secret name>=<environment variable>
```

To mount a secret as a file:

```
$ beaker session create --secret-mount <secret name>=<file path>
```

## Git Authentication

If you already use an SSH key to authenticate with GitHub, you can easily add
it to Beaker and use it in interactive sessions. First, add the SSH key to Beaker:

```
$ cat ~/.ssh/id_rsa | beaker secret write --workspace <workspace> ssh-key
```

Then, mount the SSH key into a Beaker session:

```
$ beaker session create --secret-mount ssh-key=~/.ssh/id_rsa
```

Test that the SSH key is working:

```
$ ssh -T git@github.com
```

To delete your SSH key from Beaker:

```
$ beaker secret delete <workspace> ssh-key
```

### Git Configuration

If you need to make a commit from a session, you will need to configure Git
with your name and email. You can store Git configuration in a secret an mount it into a session, just like an SSH key. On your machine, run:

```
$ cat ~/.ssh/.gitconfig | beaker secret write --workspace <workspace> git-config
```

Then create a session with your Git config:

```
$ beaker session create --secret-mount ssh-key=~/.ssh/id_rsa --secret-mount git-config=~/.gitconfig
```

## Ports

By default any open ports in your session aren't exposed on the host. Which means that if you
run something (like a webserver) that needs to listen on a port, you won't be able to access
it outside of your session.

To remedy this, you can can specify the ports you'd like exposed when you create
a new session. For instance, this exposes port `8888`, the default that's used by 
Jupyter notebooks:

```
$ beaker session create --port 8888
```

The host port that's opened won't be the same as what's requested. Instead a random, ephemeral
host port will be selected. This is both to help prevent long-running servers and prevent multiple 
sessions on the same host from making colliding reservations.

The host port that's mapped to the requested port in your container is printed out as the session
is started:

```bash
$ beaker session create --gpus 1 --port 8888
Defaulting to workspace org/workspace
Starting session ZZZ with at least 1 GPU... (Press Ctrl+C to cancel)
Waiting for session to start... Done!
Reserved 1 GPU, 15 CPUs
Exposed Ports: 0.0.0.0:49155->8888/tcp # <-- Port mapping
```

In the above example, the host port `49155` was randomly selected and bound to the port
`8888` in the interactive session. This means that connections made to port `49155` on the host will 
be routed to port `8888` in the session. So if a Jupyter notebook were running and listening
on port `8888` in the session, and the server it's running on has a network resolvable hostname of
`example.org`, we'd access the notebook by going to the URL `http://example.org:49155/` in the browser
of our choosing.

You can also see the port bindings via the `beaker session describe` command. By default the command
provides the details of the session you're currently running on a host (if there's only one). Which 
means you can execute it without any arguments, like so:

```bash
$ beaker session describe
Found running session: ZZZ
ID         ZZZ
NAME       N/A
User       Your Name
IMAGE      beaker://org/image
           https://beaker.org/im/YYY
STARTED    2022-04-27 18:13:39
ELAPSED    00:00:48
STATUS     running
GPUS       1
TCP PORTS  server.org:49155->8888/tcp (e.g., http://server.org:49155)
```

If you have multiple sessions running, you can provide the session ID as an argument:

```
$ beaker session describe <id>
```

You can bind multiple ports by specifying the `--port` flag multiple times:

```
$ beaker session create --gpus 1 --port 8888 --port 9999
Defaulting to workspace org/workspace
Starting session ZZZ with at least 1 GPU... (Press Ctrl+C to cancel)
Waiting for session to start.... Done!
Reserved 1 GPU, 15 CPUs
Exposed Ports: 0.0.0.0:49157->8888/tcp, 0.0.0.0:49156->9999/tcp
```

At this time only TCP ports can be exposed. If you need UDP support, [let us know](mailto:beaker@allenai.org).

## Reattaching to a Session

If you lose connection to a session and want to reattach to it, use `beaker session attach`.
This will connect to the running session by default. If there are multiple running sessions,
use the `--session` flag to specify one. 

## Forking a Session

Sessions can run multiple commands at once.
This can be useful if you need to check on the main process of a session without interrupting it.

To get a new terminal in an existing session, run:

```
$ beaker session exec
```

When you quit this terminal, the session will continue to run.
The session will stop when the main process exits.

You can also provide a command to `exec`. This example prints GPU utilization:

```
$ beaker session exec nvidia-smi
```

## Stopping a Session

To stop a session that you are not attached to, run:

```
$ beaker session stop
```
