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

# You can pass arguments to the build and give default values
# If you want the following to inherit your UID call the build command with:
#    docker build --build-arg build_uid=${UID} .
# If you want the following to inherit your UID and GID (if set) call the build command with:
#    docker build --build-arg build_uid=${UID} --build-arg build_gid=${GID} .

ARG build_uid=1000
ARG build_gid=1000
ARG build_username=devuser

RUN groupadd -g ${build_gid} ${build_username} && \
    useradd -r -u ${build_uid} -g ${build_gid} ${build_username}
USER ${build_username}
