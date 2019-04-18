# SET BASE IMAGE:
# Typically offered by whatever primary library/environment you wish to develop against.
FROM ubuntu:latest

# Trivial example app that has display
RUN apt update && apt install -y x11-apps

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
