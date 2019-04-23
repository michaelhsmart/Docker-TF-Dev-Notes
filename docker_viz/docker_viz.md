# docker_viz

This short example specifically focuses on getting your docker environment to
connect to your host machine's display. This is needed if the docker application
is going to create plot windows or use a GUI. Note that there is a huge caveat
with this at the bottom.

## Example container

The Dockerfile and docker-compose.yml files used for this specific example are
comparable to the basic docker_example, but have been trimmed down to just focus
on display forwarding. For a basic display application in the container we will
use `xeyes` from `x11-apps`

## XAuth

There are several ways to forward the display to the container, but we'll focus
on a way that provides some isolation. The [source
link](http://wiki.ros.org/docker/Tutorials/GUI) for this material covers some
other ways that are much easier, but also expose your host machine to greater
vulnerability.

Assuming that you already have a user configuration analogous to what was
covered in docker_example, you'll want to create an `.xauth` file that will be
mounted to the container (with the selected .xauth filename/path stored in the
`XAUTH` env variable):

```
XAUTH=/tmp/.docker.xauth
touch $XAUTH
xauth nlist $DISPLAY | sed -e 's/^..../ffff/' | xauth -f $XAUTH nmerge -
```
which [should](http://wiki.ros.org/docker/Tutorials/GUI) populate a file with
display authentication appropriate for your host user. You can then export the
XAUTH variable for later use in the session.

```
docker build . -t docker_viz:latest
...
...
docker run -it \
        --network='host' \
        --volume=$XAUTH:$XAUTH:rw \
        --env="XAUTHORITY=$XAUTH" \
        --env="DISPLAY" \
    docker_viz:latest \
    /bin/bash
```

Alternatively, you could use docker-compose:
```
version: '2'

services:
  viz_service:
    build: .
    container_name: project_viz
    network_mode: "host"
    command: /bin/bash
    volumes:
      - $XAUTH:$XAUTH:rw
    environment:
      - DISPLAY
      - XAUTHORITY=$XAUTH
      - QT_X11_NO_MITSHM=1
```
followed by `docker-compose up` (from this directory) and `docker-compose run
--rm viz_service` similar to before.

You should then be able to run `xeyes` from the container and see a pair of eyes
follow your cursor around.

If you regularly want to use a common image and display configuration, you could
consider adding `$XAUTH` to your `.bashrc` so that you don't need to constantly
re-create/export it for use with your containers.

## Resources

I used these links:
- http://wiki.ros.org/docker/Tutorials/GUI
- https://dzone.com/articles/docker-x11-client-via-ssh
- https://skandhurkat.com/post/x-forwarding-on-docker/

## Extension to Remote

If you're insterested, a lot of these ideas extend to working with remote
machines as well. Some people do all of their work "in the cloud" and do
development off of a light notebook that just serves as a GUI front-end. That's
well out-of-scope here but is still neat to know about.

## Caveat: This is Not The Normal Docker Pattern

The typical idea with docker is that each container will host ONE app. Running
GUI apps within a developer context within a container leads to the very obvious
question of "Can I just put my IDE in a container?". The answer to this is yes,
but you would then have several processes in the container and you might as well
consider using a full VM instead as docker isn't designed with this in mind.

Instead of directly using the display, many apps use port forwarding to achieve
their goals. For example, compare this to the tensorflow container's JuPyter
example. That example starts JuPyter and forwards the appropriate port to the
host so that you can then effectively work with a GUI via your browser.

In this same vein, a better way to get your IDE working against the container
environment, as supported by PyCharm, is to have the IDEpoint to the interpreter
within a container environment. That is covered in the docker_pycharm example.

## Known Issues

An "MIT-SHM" error is quite common as shown [here, at the
bottom](http://wiki.ros.org/docker/Tutorials/GUI). The linked solution is to
include the environment variable `QT_X11_NO_MITSHM=1` either in the docker run
command or in the docker-compose.yml file depending on your execution method.
