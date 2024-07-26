# [Yocto] 01_Building Yocto Images using a Docker Container

Building a Yocto image requires a specific set of Operating System dependencies that might make it challenging to run on a non-dedicated machine. Over time it can be difficult to maintain a known-good environment that will work with a number of different versions of the Yocto project. One solution to this is to use a Docker container to create a known-good build environment, containing the necessary packages and Operating System version.

# 1. Introduction

The creation of a Docker container can be achieved in a number of ways. This specific guide shows how to install the bare minimum infrastructure to build Yocto images for NXP hardware while storing the generated build artifacts outside of the container itself. The Docker container will mount local folders on the host machine, and no persistent data will be stored within the Docker container filesystem. This has the advantage of not requiring containers to be part of a backup strategy, and allows for a very lightweight build environment.

# 2. Installation of Docker Engine on a Linux machine

The process to install Docker on a host Linux machine can vary based on the operating system that is being run natively. Please follow the instructions at [Install Docker Engine](https://docs.docker.com/engine/install/) to install the Docker engine. Be sure to also follow the post-installation steps at [Linux post-installation steps for Docker Engine](https://docs.docker.com/engine/install/linux-postinstall/) in order to allow users to run Docker without root/sudo access. The remaining sections assume that this has all been completed successfully, and as a user, you are able to successfully run the “hello world” docker test image:

```console
$ docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete
Digest: sha256:4bd78111b6914a99dbc560e6a20eab57ff6655aea4a80c50b0c5491968cbc2e6
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
```

# Building a Docker Image

On your host machine, create a new directory, and and create the following contents as a file “Dockerfile”. 

https://github.com/carloscn/dockerhub/tree/master/ubuntu1604

```Dockerfile
FROM ubuntu:16.04
MAINTAINER Carlos Wei <calos.wei.hk@gmail.com>

ARG DEBIAN_FRONTEND=noninteractive
RUN mv /etc/apt/sources.list /etc/apt/sources.list.bak

RUN \
    echo "deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted universe multiverse" > /etc/apt/sources.list && \
    echo "deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse" >> /etc/apt/sources.list && \
    echo "deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse" >> /etc/apt/sources.list && \
    echo "deb http://security.ubuntu.com/ubuntu/ xenial-security main restricted universe multiverse" >> /etc/apt/sources.list

RUN \
    dpkg --add-architecture i386 && \
    apt update && \
    apt install locales apt-transport-https ca-certificates curl sudo vim -y && \
    apt install tofrodos iproute2 gawk xvfb gcc git make net-tools libncurses5-dev zsh \
    tftpd zlib1g-dev libssl-dev flex bison libselinux1 gnupg wget diffstat chrpath socat \
    autoconf libtool tar unzip texinfo gcc-multilib build-essential libsdl1.2-dev libglib2.0-dev  \
    libssl-dev screen pax gzip vim net-tools cmake  android-tools-adb android-tools-fastboot      \
    autoconf  automake bc bison build-essential ccache codespell cscope curl device-tree-compiler \
    expect flex ftp-upload gdisk libattr1-dev libcap-dev libfdt-dev libftdi-dev libglib2.0-dev   \
    libgmp-dev libhidapi-dev libmpc-dev libpixman-1-dev libssl-dev libtool make mtools \
    netcat ninja-build python-crypto python3-crypto python-pyelftools ntpdate \
    python3-pyelftools rsync unzip uuid-dev xdg-utils xz-utils lzop cpio \
    zlib1g-dev proxychains libqt5qml5 libqt5quick5 libqt5quickwidgets5 qml-module-qtquick2 \
    libgsettings-qt1 xinetd tftp-hpa aria2 minicom guake openssh-client openssh-server libncursesw5 -y && \
    rm -rf /var/lib/apt-lists/* && \
    echo "dash dash/sh boolean false" | debconf-set-selections && \
    dpkg-reconfigure dash

RUN curl https://storage.googleapis.com/git-repo-downloads/repo > /bin/repo && chmod a+x /bin/repo
RUN sed -i "1s/python/python3/" /bin/repo
RUN groupadd build -g 1000
RUN useradd -ms /bin/bash -p build build -u 1028 -g 1000 && \
        usermod -aG sudo build && \
        echo "build:build" | chpasswd
RUN echo "en_US.UTF-8 UTF-8" > /etc/locale.gen && \
        locale-gen
ENV LANG en_US.utf8
USER build
WORKDIR /home/build
RUN git config --global user.email "build@example.com" && git config --global user.name "Build"
```

https://github.com/carloscn/dockerhub/blob/master/ubuntu1604/Dockerfile

This Dockerfile shall create a base Ubuntu 16.04 OS image, and install the necessary packages and additional tools required to build IMX6 Yocto Images. A user “build” is also created within the container, and shall be used for the build process.

Build your Docker image using the following command, making sure you are in the directory containing the Dockerfile. You can choose a different name to “yoctocontainer” if you prefer. Ensure you also change the name appropriately when running the docker run command if you do.

`docker build -t yoctocontainer .`

Building will take around a minute, depending on bandwidth to download the dependencies. On subsequent builds, downloads will be cached and so will be substantially quicker. The build command only needs to be run once, unless you modify the Dockerfile to include new packages or tools.

Once the image is created, you can change to any directory on your host. There is now no dependency on the Dockerfile being in the current directory (or even existing), as the configuration of “yoctocontainer” is centrally managed on your machine.

The docker run command can now be executed. The -v options below share folders between the host and the docker image. This is important to note as the docker container will be run in a mode where no data local to the docker machine will survive a relaunch!

As a result, the example below uses the current working directory (pwd) and mounts it as a folder _/home/build/work_. As long as you change directory and run all of your Yocto commands as a folder within work, this will remain on your target system after docker has finished. Feel free to add additional paths to the example shown below, and remove unused paths:

```bash
docker run \
--device=/dev/kvm:/devb/kvm \
--device=/dev/net/tun:/dev/net/tun \
--cap-add NET_ADMIN \
--hostname buildserver \
-it \
-v /tftpboot:/tftpboot \
-v `pwd`:/home/build/work \
yoctocontainer
```

https://github.com/carloscn/dockerhub/blob/master/ubuntu1604/run_your_build_docker.sh

>This Docker image is purposefully minimal - it is intended to be a build environment only! This means, you will not find common editing tools such as vi or nano, as files should be modified on the host, and the docker image used to build the image. Always work in a mounted filesystem (such as _~/build/work_) to ensure that your data persists across sessions!

Once completed, you can quit back to your host shell using the exit command. The next invocation of the container will be “fresh” as no data persists across runs, outside of that mounted from the host Operating System.

# Storing the Docker Image your built


![](https://raw.githubusercontent.com/carloscn/images/main/typoratyporatypora202407100838814.png)


```console
# carlos @ asus_carlos in ~/storage [8:38:33] 
$ docker ps -a
CONTAINER ID   IMAGE                         COMMAND       CREATED        STATUS                    PORTS     NAMES
9875b7bc364f   yoctocontainer                "/bin/bash"   2 hours ago    Up 2 hours                          stupefied_cori
415aa304458a   carloswei/ubuntu1804_emb:v1   "/bin/bash"   24 hours ago   Up 24 hours                         quizzical_franklin
20041bc19285   carloswei/ubuntu2004_emb:v1   "/bin/bash"   25 hours ago   Exited (0) 25 hours ago             temp_v2

# carlos @ asus_carlos in ~/storage [10:17:26] 
$ docker commit -a 'Carlos Wei' -m "yocto built finished" 9875b7bc364f yoctocontainer:v1
sha256:69c437f498c45ab88f6e9a712ba36d5008ebe7c5e1631bea2ffd773e06be9279

# carlos @ asus_carlos in ~/storage [10:18:32] 
$ docker images
REPOSITORY                 TAG       IMAGE ID       CREATED         SIZE
yoctocontainer             v1        69c437f498c4   6 seconds ago   1.05GB
yoctocontainer             latest    9a3a326304a7   2 hours ago     1.04GB
carloswei/ubuntu2404_emb   v1        2be5a661285e   46 hours ago    5.7GB
carloswei/ubuntu2204_emb   v1        ae73c0d02a2f   46 hours ago    5.6GB
carloswei/ubuntu2004_emb   v1        37ba311ea955   47 hours ago    5.51GB
carloswei/ubuntu1804_emb   v1        a9b6f5e2daf5   47 hours ago    5.31GB
carloswei/ubuntu1604_emb   v1        2e6eea371ac2   2 days ago      5.29GB
```


# Reference

* https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/2823422188/Building+Yocto+Images+using+a+Docker+Container