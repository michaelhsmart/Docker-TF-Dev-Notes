# docker

Docker is an extremely powerful tool that has seen significant use in web-dev,
but is also seeing use in ML lately, specifically in the area of GPU
utilization. Both Tensorflow and NVIDIA have significant docker support and
streamlining, while our DGX-style clusters require docker for use.

Docker containers wrap up software with its dependencies into a standalone unit
that can be executed. Docker containers have a lot of similarities with virtual
machines, but they are very different. Instead of the hardware being virtual, it
is the OS layer that is virtual. Docker then communicates directly with the
kernel, allowing the container to run as though not only different system
dependencies may be installed, but the OS itself may be different! It's this
feature that we've actually been using for years to test the `autonomoose` stack
on Travis-CI even though the Travis machine only had Ubuntu 14.04 itself.

A key benefit that you might recognize with this level of isolation is that if
the container's OS is completely separated, then your host machine is largely
protected from whatever mistakes you might make within the container's OS. You
can't break your local installation if you can't touch it. "My install script
broke my system installation of XYZ so I had to reinstall Ubuntu as that was
just the easiest way to fix it" can become a memory of the past. Unless you're
lucky enough to have never had such an experience, you can see the benefit of
this.

Lastly, specifically where nvidia/CUDA are concerned, working with a library
with CUDA dependencies is made much easier with docker and nvidia's derived
[nvidia-docker](https://github.com/NVIDIA/nvidia-docker). This isn't an
exaggeration. My local machine's setup is currently quite broken due to an
errant setup script, but the virtualization provided by docker means I haven't
needed to bother fixing it yet.

## Resources

There are a lot of resources for learning about docker. I specifically find the
following helpful:

- [Tensorflow & Docker](https://www.tensorflow.org/install/docker)
- [A Beginner's Guide to Docker](https://medium.freecodecamp.org/a-beginners-guide-to-docker-how-to-create-your-first-docker-application-cc03de9b639f)
- [Docker Get Started](https://docs.docker.com/get-started/) (Sections 1-3)
- [Best Practicies for writing
  Dockerfiles](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Docker Compose](https://docs.docker.com/compose/)
- [Docker Volumes](https://docs.docker.com/storage/volumes/)

There's also tons of youtube videos on it if that's more your thing (my personal
favourites):
- [Learn Docker in 12 Minutes](https://www.youtube.com/watch?v=YFl2mCHdv24)
- [Docker Compose in 12 Minutes](https://www.youtube.com/watch?v=Qw9zlE3t8Ko)

Or a longer/slower course:
- [HowToCode's Docker Course](https://www.youtube.com/watch?v=tmyFd1PD-Gs)

## [nvidia-docker](https://github.com/NVIDIA/nvidia-docker)

nvidia-docker is a distinct addition to the docker ecosytem that has its own
installation that sets up the CUDA drivers/paths/variables for dependent
containers. Once it's installed, all you need to do is specify the
`--runtime=nvidia` argument in the `docker run` command to ensure that it is
Nvidia's container runtime being called instead of the default Docker runtime:

```
# Test nvidia-smi with the latest official CUDA image
msmart@msmart-desktop:~$ docker run --runtime=nvidia --rm nvidia/cuda:10.1-base nvidia-smi
Unable to find image 'nvidia/cuda:10.1-base' locally
10.1-base: Pulling from nvidia/cuda
898c46f3b1a1: Pull complete
63366dfa0a50: Pull complete
041d4cd74a92: Pull complete
6e1bee0f8701: Pull complete
131dbe7c254d: Pull complete
5bca6b05dcd6: Pull complete
0d286a7b6e12: Pull complete
Digest: sha256:6ddf907e77f4b53ac8b0b8ce9fa9cd43ffb6882f1ad0f2d41ca996f154f17c7b
Status: Downloaded newer image for nvidia/cuda:10.1-base
Thu Apr 11 22:00:11 2019       
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 418.40.04    Driver Version: 418.40.04    CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  TITAN Xp            Off  | 00000000:01:00.0  On |                  N/A |
| 23%   28C    P8    12W / 250W |   1079MiB / 12192MiB |      3%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
+-----------------------------------------------------------------------------+
```

where it is visible that while my GPU is performing work (such as my display),
those processes can't be seen from the container.

### Editing docker daemon.json

If the installer hasn't already done so, you may wish to update
`/etc/docker/daemon.json` to set NVIDIA as the default docker container runtime:

```
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```
which then frees you from having to provide the `--runtime=nvidia` argument to
`docker run`, i.e.:
```
msmart@msmart-desktop:~$ docker run --rm nvidia/cuda:10.1-base nvidia-smi
```

### Caveat: Docker vs virtualenv for Python

An important distinction is that while you _can_ use docker to achieve most of
the same isolation objectives as what `virtualenv` allows, it is important to
recognize that they are fundamentally different. There's also nothing stopping
you from using a `virtualenv` inside a docker container, and I'm sure there's
likely some places where that will be appropriate. They're trying to do two
separate things.

The main mechanism behind `virtualenv` is to just recognize that most of the
potential problems with package conflicts flow through the `PATH` and
`PYTHONHOME` environment variables. If you manage these variables properly, then
you can manipulate applications to looking in the desired place for the
interpreter and  dependencies that they need. The other dependency versions may
still be on your machine, but adjusting these environment variables prevents
your application from finding them. On the other hand, the Docker approach to
this problem creates a virtualization that only has the desired versions in the
first place so that there is no undesired version for the application to find.
Further, Docker's isolation goal separates much more than just the application
dependencies. You could still thoroughly wreck your setup from within an
activated virtualenv, whereas the host machine has much more protection from
code within a docker container. Docker's isolation of the Python environment is
simply a consequence of its larger aim to isolate pretty well everything.

I find this article useful for distinguishing the two through the case where one
may wish to use a `virtualenv` within a docker container:
https://pythonspeed.com/articles/activate-virtualenv-dockerfile/

### Warning: Delete Your Containers / Monitor Docker's Space Consumption

A key aspect of docker's ideology is that containers should be ephemeral and
should support being deleted/recreated at will - meaning that they do not store
persistent state internally. It's common to use the `--rm` argument with `docker
run` to reflect this: it deletes the container after the process ends. You would
then create a new fresh container from the built image the next time you run the
desired docker command. This "always fresh" idea is central to the docker ethos.
If you forget the `--rm` option, or if you decide to re-use containers, be aware
that the number of containers stored on your machine may grow and may take up
considerable memory if you never flush them out. The commands: `docker container
purge` and `docker image purge` can remove outdated containers and images from
your system. If they're insufficiently effective for you, you can google other
ways to clean it out. Just be aware of this as a thing and periodically check
your docker space consumption.
