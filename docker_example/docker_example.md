# docker_example

## Basic Example

If you've already completed the setup instructions for Tensorflow via Docker (https://www.tensorflow.org/install/docker), then making your own environment
for experimenting with Tensorflow in a way that is isolated from your other work
is straightforward.

If you want to play around in JuPyter with a specified version of Tensorflow,
you can set that up directly from the command line (_warning_: we'll get to
volume mounting that allows your notebooks to be saved on your host machine
later):
```
docker run -it -p 8888:8888 tensorflow/tensorflow:1.13.1-gpu-py3-jupyter
```

What if the main docker container doesn't have a desired dependency?

For example:
```
msmart@msmart-desktop:~$ docker run -it tensorflow/tensorflow:1.13.1-gpu-py3
...
...
root@7544b47f417b:/# python
Python 3.5.2 (default, Nov 12 2018, 13:43:14)
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import matplotlib
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
ImportError: No module named 'matplotlib'
```

If you want to work on a library or make modifications to your container's
environment, such as add additional dependencies, you can create a new
Dockerfile to do so. This example Dockerfile contains:

```
# SET BASE IMAGE:
# Typically offered by whatever primary library/environment you wish to develop against.
FROM tensorflow/tensorflow:1.13.1-gpu-py3

# Perform any extra setup you need, or run a setup script for your project

# Need this in order to resolve an opencv installation issue within container
RUN apt update && apt install -y \
    libsm6 \
    libxext6 \
    libxrender-dev

# Install python packages within container
ENV PYTHON_PACKAGES="\
    matplotlib \
    numpy \
    opencv-python \
    pillow \
    pypng \
    pyyaml \
    scipy \
    cython \
"
RUN pip install --no-cache-dir $PYTHON_PACKAGES
```

Here, we decide that our project's containers will be derived from
`tensorflow/tensorflow:1.13.1-gpu-py3` and we then specify other
installation/setup steps the environment requires. You could use
`tensorflow/tensorflow:1.13.1-gpu-py3-jupyter` as the base if you wished (or
whatever version of tensorflow is appropriate).

You can then build the image from the directory containing the Dockerfile,
tagging it as the `latest` version of `project_foo`:
```
docker build . -t project_foo:latest
```

Now we can try to find matplotlib in a container built from the new image:

```
msmart@msmart-desktop:~$ docker run -it project_foo:latest
...
...
root@0f66cd3330eb:/# python
Python 3.5.2 (default, Nov 12 2018, 13:43:14)
[GCC 5.4.0 20160609] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import matplotlib
>>> help('matplotlib')
Help on package matplotlib:

NAME
    matplotlib - This is an object-oriented plotting library.
...
```

where we now see success.

## Volumes

A key design idiom with Docker is that containers are meant to be *ephemeral*.
Screw something up in your container? Delete it and make a new one. Finished
running a container's process? Save on cleanup effort by just deleting it, or
even better by using the `--rm` argument with `docker run` to delete the
container automatically once you're done running it. What about data that we
wish to be persistent, or shared? The answer is
[volumes](https://docs.docker.com/storage/volumes/).

Volumes allow you to mount directories to be shared between your host machine
and container, either as read-only or read-write. They can be mounted at
container runtime with `docker run` through the `-v` argument. For example, to
give a container readonly access to my Datasets directory, and rw access to both
my project source directory and a scratch directory, I can use:
```
$ docker run -it \
  --name volume_example \
  -v /home/msmart/Datasets:/Datasets:ro \
  -v /home/msmart/Project_Foo/src:/app \
  -v /home/msmart/Job_123_Scratch:/scratch \
  tensorflow/tensorflow:1.13.1-gpu-py3
```
and see these directories:
```
root@a86a5b3e733d:/# ls
Datasets  bin   dev  home  lib64  mnt  proc  run   scratch  sys  usr
app       boot  etc  lib   media  opt  root  sbin  srv      tmp  var
```

now if I try (this is just an illustrative example - do not get too reckless!):
```
root@a86a5b3e733d:/# rm -rf /Datasets/
```
I get a flood of:
```
...
rm: cannot remove '/Datasets/coco/unlabeled2017/000000076141.jpg': Read-only file system
rm: cannot remove '/Datasets/coco/unlabeled2017/000000076175.jpg': Read-only file system
rm: cannot remove '/Datasets/coco/unlabeled2017/000000076180.jpg': Read-only file system
rm: cannot remove '/Datasets/coco/unlabeled2017/000000076197.jpg': Read-only file system
```

well, that's a relief!

## User Permissions

You may have noticed either from warnings or from the shell prompt within the
docker container that the default user configuration within a docker container
is typically root. Since the container often has isolated resource access, this
usually isn't a huge problem. What if you want some shared resource access
between your host machine and the container? An example shared resource could be
your local host's display driver if you want the container to display
windows/figures, or it could be a scratch directory on your machine where you
don't want the application to create all of its files with root permissions.

Tensorflow's containers usually contain the following warning about this on run:
```
WARNING: You are running this container as root, which can cause new files in
mounted volumes to be created as the root user on your host machine.

To avoid this, run the container by specifying your user's userid:

$ docker run -u $(id -u):$(id -g) args...
```

If we add `-u $(id -u):$(id -g)` to our docker run arguments for a tensorflow
container, we then see something like:

```
You are running this container as user with ID 1000 and
group 1000, which should map to the ID and group for your user on the Docker
host. Great!
```

It is also possible to specify the container's user in a
[Dockerfile](https://docs.docker.com/v17.09/engine/reference/builder/#user)
For example:
```
# Rest of dockerfile as above
RUN useradd -r -u 1000 -g 1000 devuser devuser
USER devuser
```
which would save you from needing to specify the `-u` argument for docker run,
and save you the risk of forgetting and accidentally using a container as root.
If you have a different UID/GID than 1000, or want it to be configurable, you
could declare them as ENV variables that you can export and use in the
dockerfile. The **key** advantage of this approach is that it will also add the
new user's $HOME directory within the container. Some applications may require a
$HOME in order to function. This issue should be distinct from having hard-coded
paths, however.

Without specifying the user, files or directories added to the host computer via
a volume shared with the container could have their permissions set with root.

User permissions with docker can actually get quite complicated, but this
article does a great job covering it:
https://medium.com/@mccode/processes-in-containers-should-not-run-as-root-2feae3f0df3b

We'll need to revisit this topic in greater depth at a later date, but for now
this should be sufficient for the task of basic development.

## Docker Compose

Based on the previous section, our run command may now look something like:

```
$ docker run -it \
  --name project_foo_dev \
  -v /home/msmart/Datasets:/Datasets:ro \
  -v /home/msmart/Project_Foo/src:/app \
  -v /home/msmart/Job_123_Scratch:/scratch \
  -u $(id -u):$(id -g) \
  project_foo:latest
```

and we haven't even added JuPyter or Tensorboard port specifications, or display
forwarding! Also, this just runs a single container and you might have other
processes you wish to run in different containers. This command can clearly grow
to be unworkable.

You might be thinking of writing your own bash-script or alias to control and
manage this, but Docker has its own framework for composing `docker run`
commands and services together: [docker
compose](https://docs.docker.com/compose/gettingstarted/) (requires separate
installation)

An example compose file could look something like this:
```
version: '2'

services:
  dev_interpreter:
    container_name: project_foo_dev
    command: /bin/bash
    image: project_foo:latest
    volumes:
      - ~/Datasets:/Datasets:ro
      - ~/Project_Foo/src:/app
      - ~/Job_123_Scratch:/scratch
    environment:
      - Dataset_Dir=/Datasets
      - Src_Dir=/app
      - Scratch_Dir=/scratch
```
with a filename of `docker-compose.yaml`. This will mount three volumes into the
container, and set 3 environment variables in case the container needs to be
told where those directories are.

You can then use `docker-compose up` to run the composition, and then you can
use `docker-compose run --rm dev_interpreter` (`dev_interpreter` is the name of
the service defined in the compose file) and get into a bash session with the
volumes mounted. When finished you can run `docker-compose down`. I recommend
reading Docker's tutorials: https://docs.docker.com/compose/gettingstarted/

#### Known Issues

- Volume Creation: If you list a volume in the docker-compose.yml file that
corresponds to a non-existant directory on the host machine, then
`docker-compose up` will create it *as root*. The easiest way to avoid this is
to make sure that those directories already exist, or to downgrade their
permissions after.

#### Environment Variables

If you're looking to leverage environment variables from your shell session or
bashrc within docker-compose, check these links:

- https://docs.docker.com/compose/environment-variables/

- https://docs.docker.com/compose/compose-file/compose-file-v2/#variable-substitution

Specifically note that:
> Compose uses the variable values from the shell environment in which docker-compose is run.

Also, this:
https://unix.stackexchange.com/questions/3507/difference-between-shell-variables-which-are-exported-and-those-which-are-not-in

Lastly, you can add a .env file to declare ENV variables that `docker-compose
up` will evaluate: https://docs.docker.com/compose/env-file/
