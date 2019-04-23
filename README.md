# Docker-TF-Dev

This repo contains on-boarding docs/examples for setting up research
projects/pipelines with docker and tensorflow, jupyter etc:

- `docker_basic/` contains a quick and trivial example for the baseline task
  of running some code with tensorflow on your local GPU without actually
  installing tensorflow (or even the CUDA libraries!). Also goes over some
  deeper detail involved in using docker for the basic dev setup.
- `docker_viz/` contains an example and references for running a docker session
  that requires access to your local machine's display (which is not accessible
  by default)
- `docker_pycharm/` contains an example and references for using Pycharm for
  development, but having the iterpreter reside within a docker containers

# README

## Motivating Scenarios

Before diving into the specific tools for developing our code and research
projects, let's take a step back and consider some high-level example situations
we may commonly run into. What kind of work do we do and what kind of
development environment properties do we care about? We'll see later that there
are plenty of tools that can help us achieve our different goals in each of the
phases.

#### Early Experimentation

In early experimentation, maybe you want to throw together ~100 lines of Python
in a single session and see what happens, or test out a new library. The main
concerns here are that the tooling is lightweight but also light in impact. You
don't want anything you hack together to affect any of your longer-term work or
force you to reinstall anything later on.

#### Serious Development

When you want to more seriously develop an idea, either as its own application
or as a thoroughly fleshed out library, you want to have access to all of the
benefits of an IDE for debugging/static analysis. It can also be critical to
establish a common and controlled working environment during this process,
especially if you're working on a team. There's lots of great IDE's out there,
but we tend to use the JetBrains family (PyCharm/CLion)

#### Demonstration

Once your project has been developed, you likely want to build a demo to show it
off or to generate example results. Demos should be specific about what they
want to show and can hide the other machinery in the back-end to avoid
distraction.

#### Deployment

If you want to share/relocate your software, either to train on a DGX-style
server, or to distribute for other researchers to try, you'll need to make sure
that the environment and all of its dependencies are also accounted for in that
transfer. You also want to script literally everything such that you can reduce
the process to a single line if possible (that single line can just call a few
independent scripts in sequence). If you're deploying on a server, you might not
have any interactivity with your training job, so you might need it to operate
independently. For other researchers trying out your code, you really want to
make sure that your setup script doesn't conflict with their unknown system as
well.

## Tools

Each of the motivating scenarios have tools that help resolve issues you might
run into. We'll dive into the tools in more detail later in their own files, but
my resulting recommendation is:

Overall: Docker for general environment management (see `docker.md` for an
overview).
- Early Experimentation: Jupyter Notebook or PyCharm
- Serious Development: PyCharm Professional for building libraries and tests
- Demonstration: Jupyter Notebook
- Deployment: bash scripts within docker

The different tools for different tasks can also help as a cue for file
separation as well. If you are doing all of your experimenation, library
development, and results generation in overlapping files, then you're going to
have a rough time.

For examples of some repos I like in this regard check out:
* https://github.com/cocodataset/cocoapi/tree/master/PythonAPI
* https://github.com/matterport/Mask_RCNN

## Comments on the Examples

The examples I provided assumed projects located elsewhere, but you can add the
relevant Dockerfile or docker-compose.yml files to your own projects. They could
then be contained within your project's version control. In that case, you may
want to change some of the hardcoded options to environment variables so that
your setup is more portable.
