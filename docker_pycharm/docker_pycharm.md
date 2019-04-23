# docker_pycharm

If you can run source code in a container, and you can get display windows out
of containers, then the last thing we might for our ML dev pipeline is to be
able to use a proper IDE with a container for developing the libraries that form
the back-ends of our research applications. This can be accomplished using
PyCharm Professional (which students can get for free, and isn't too expensive
for everybody else when you consider how useful it can be). We'll essentially be
combining docker_example and docker_viz with some PyCharm features.

## Option 1 - The Brute Force Approach (Not Ideal)

While against the overall docker ethos, it is completely possible to run your
IDE within a docker container. If you do this, perform the PyCharm specific
Dockerfile steps after your main dev image steps so that most of the image can
simply reuse a basic development image. Note, that if you set the container user
before installation steps, the user might not have permissions to finish
building the image. This approach works for PyCharm Community edition as well.

This example uses the files in the subdirectory `example_1`.

First you need to add java and some other PyCharm dependencies to the container:
```
RUN apt update && apt install -y libxtst6 \
   libxi6 \
   default-jre
```

If you then add these lines (adjusting for your pycharm tar name and version):
```
ADD pycharm-community-2019.1.1.tar.gz /opt/
CMD /opt/pycharm-community-2019.1.1/bin/pycharm.sh
```
then PyCharm will run on `docker run` of the resulting container (assuming the
PyCharm tar is in the current directory). If you mount your PyCharm config
directory to the container, then you will be able to use an existing
configuration on installation - provided your usernames match.

This again becomes easier with docker-compose, as you can instead mount the
extracted pycharm folder from whatever directory you like and specify the
various volume mounts. See example_1 for example files. These examples assume
you have exported ${DOCKER_BUILD_UID} and ${XAUTH_DOCKER} environment variables
in the current shell session (you could add them to your .bashrc).

- **NOTE**: In theory you could look at forwarding your host installation's
  configs or even the host's PyCharm installation itself, but I won't venture
  there

## Option 2 Setting the PyCharm Interpreter (Much Better)

By default, PyCharm uses its own internally managed `python` interpreter, which
may not play well with your local machine's default versions and virtual
environments depending on your application. Fortunately, PyCharm Professional
allows you to resolve this issue by specifying the python interpreter you wish
it to use.

Relative to Docker, you can either point to an existing (already built) docker
image, or to a docker-compose file.

- Image: https://www.jetbrains.com/help/pycharm/using-docker-as-a-remote-interpreter.html
- ~~Compose: https://www.jetbrains.com/help/pycharm/using-docker-compose-as-a-remote-interpreter.html~~

**TODO**: I have not yet managed to get the combination of PyCharm +
docker-compose + nvidia-docker's runtime working. If you try and can get it
working, please let me know.

**NOTE**: When PyCharm initially connects against a remote interpreter, it has
to run through its full skeleton of static analysis against that interpreter.
This can take a very long time, but doesn't appear to need re-runs if the
run/debug configuration is not changed.

### Initial Advice

I strongly recommend you debug your Docker environment directly before
moving on to using it to host the remote interpreter for PyCharm. Docker
problems are much harder to deal with through PyCharm than directly.

### Dockerfile

This example uses the files in the subdirectory `example_2`.

0. Build this image with `docker build . -t docker_pycharm:latest`:
1. Create a an empty project folder ~/Project_Foo
2. Open PyCharm and open that folder as an existing project. NOTE: There is
currently a PyCharm limitation that prevents creating a new project specifically
against a new Docker Interpreter:
https://intellij-support.jetbrains.com/hc/en-us/community/posts/115000636990-PyCharm-This-interpreter-type-does-not-support-remote-project-creation
3. Create a new file (bar.py) containing:
```
import tensorflow as tf
hello = tf.constant('Hello, TensorFlow!')
sess = tf.Session()
print(sess.run(hello))
```

You should notice PyCharm complaining about not having the appropriate packages
when you run `bar.py` (assuming you don't have a native installation).
```
/usr/bin/python3.5 /home/msmart/Project_Foo/bar.py
Traceback (most recent call last):
  File "/home/msmart/Project_Foo/bar.py", line 1, in <module>
    import tensorflow as tf
ImportError: No module named 'tensorflow'

Process finished with exit code 1
```
4. Go to Settings->Project Settings->Project Interpreter->Gear Icon->Add
5. Select the `docker_pycharm:latest` image, click Apply
6. Observe that PyCharm's in-code complaints are gone!
7. Run `bar.py` and observe that the code runs in the container

```
7b67186e700d:python -u /opt/project/bar.py
2019-04-16 21:29:15.313522: I tensorflow/core/platform/cpu_feature_guard.cc:141] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX2 FMA
2019-04-16 21:29:15.434960: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:998] successful NUMA node read from SysFS had negative value (-1), but there must be at least one NUMA node, so returning NUMA node zero
2019-04-16 21:29:15.435931: I tensorflow/compiler/xla/service/service.cc:150] XLA service 0x59fd060 executing computations on platform CUDA. Devices:
2019-04-16 21:29:15.435943: I tensorflow/compiler/xla/service/service.cc:158]   StreamExecutor device (0): TITAN Xp, Compute Capability 6.1
2019-04-16 21:29:15.445797: I tensorflow/core/platform/profile_utils/cpu_utils.cc:94] CPU Frequency: 3696000000 Hz
2019-04-16 21:29:15.446400: I tensorflow/compiler/xla/service/service.cc:150] XLA service 0x5a667a0 executing computations on platform Host. Devices:
2019-04-16 21:29:15.446414: I tensorflow/compiler/xla/service/service.cc:158]   StreamExecutor device (0): <undefined>, <undefined>
2019-04-16 21:29:15.446794: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1433] Found device 0 with properties:
name: TITAN Xp major: 6 minor: 1 memoryClockRate(GHz): 1.582
pciBusID: 0000:01:00.0
totalMemory: 11.91GiB freeMemory: 10.57GiB
2019-04-16 21:29:15.446807: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1512] Adding visible gpu devices: 0
2019-04-16 21:29:15.447966: I tensorflow/core/common_runtime/gpu/gpu_device.cc:984] Device interconnect StreamExecutor with strength 1 edge matrix:
2019-04-16 21:29:15.447977: I tensorflow/core/common_runtime/gpu/gpu_device.cc:990]      0
2019-04-16 21:29:15.447982: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1003] 0:   N
2019-04-16 21:29:15.448259: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1115] Created TensorFlow device (/job:localhost/replica:0/task:0/device:GPU:0 with 10280 MB memory) -> physical GPU (device: 0, name: TITAN Xp, pci bus id: 0000:01:00.0, compute capability: 6.1)
b'Hello, TensorFlow!'

Process finished with exit code 0
```

You may have noticed that PyCharm will by-default mount your project in
`/opt/project/`, this can be configured in your run/debug configurations.

You can also configure PyCharm's python console to also run against the chosen
container.

### Configurations - Run/Debug Configurations

If you look at the `Docker container settings` field, you'll see where the
`/opt/project/` mount comes from.  If you edit these settings, then you'll see
all of the main options we would have previously set as arguments to `docker
run`, including the ones used for display binding.

Note that you won't need to mount the project's source directory explicitly as
PyCharm will mount the source directory automatically so that the container's
python interpreter will be able to run/debug the code. Of course, you can modify
the mounted location from `/opt/project/` if you want.

Let's extend the `bar.py` script to load and show an image from the KITTI object
dataset, which I have locally at `~/Datasets/KITTI/object`

We need to add the volumes (edit run/debug configurations->docker container
settings) as bindings:
* Container path: `/Datasets/KITTI/object` Host path: `~/Datasets/KITTI/object`
  Read-Only
* Container path: `/tmp/.docker.xauth` Host path: `/tmp/.docker.xauth`

with the second mount allowing the container to write to the `.xauth` file
covered in the `docker_viz` example.

We'll also need to set a bunch of environment Variables:
* `QT_X11_NO_MITSHM=1`
* `DISPLAY=:0` - or to whatever `echo $DISPLAY` gives you
* `XAUTHORITY=/tmp/.docker.xauth`
* `DATASET_DIR=/Datasets` so that within the container, `$DATASET_DIR` will
give the location of the datasets folder.

We then set the network to host. Note that if we wanted our project to be able
to output artifacts (like network weights) we could mount a specific directory
for them to be placed and indicate it to the container with an environment
variable.

Then, if you update `bar.py` as follows, you should be able to run it from
PyCharm against the container and see an image on your host machine!
```
import cv2
import os

import tensorflow as tf

hello = tf.constant('Hello, TensorFlow!')
sess = tf.Session()
print(sess.run(hello))

datasets_dir = os.environ['DATASET_DIR']
img = cv2.imread(datasets_dir + '/Kitti/object/training/image_2/000000.png', cv2.IMREAD_COLOR)

cv2.imshow('image', img)
cv2.waitKey(0)
cv2.destroyAllWindows()
```

Lastly, on the main page for editing this specific run/debug configuration, if
you check the "Share" box, this configuration will be exported as a `*.xml` file
under the `<project_directory>/.idea/runConfigurations`. i.e. on my machine this
configuration exports to:
`/home/msmart/Project_Foo/.idea/runConfigurations/bar.xml`

you could then share/copy the config as needed.

See the accompanying image for what the PyCharm docker container config looks
like.

## Version Control Your Dockerfile

In this example, I just used the included Dockerfile and gave the image an
arbitrary `docker_pycharm:latest` tag. You should consider adding a
version-controlled Dockerfile to your project that can then be shared for both
development and deployment. You could build and tag the image via an install
script. So long as the user has nvidia-docker set up with the supporting driver
version, setup could be both one-line and much safer!
