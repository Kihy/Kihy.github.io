---
layout: post
title:  "Setup Deep Learning Environment"
author: "Ke He"
description: "Using Docker to set up a stable and distributable deep learning environment"
---

I have been a fan of using dockers to set up my deep learning environment and running all my code in docker. The advantages of docker is that it is stable, isolated and can be easily distributed to someone else with either docker hub or tar files. Docker works in layers, where each layer is stacked on top of another, with base layer as parent. Each layer and their parents can be thought of a VM and can be run on its own. It is also possible to map local files and folders into docker environment so that you don't need to start docker to see the files you generated.

All my previous work was done in docker environment, but recently my docker images seems to be disappearing after restarting my ubuntu distro for some reason, so I removed everything and reinstalled it(Removing docker does not automatically remove containers and images, which saves a lot of time for reinstallations).

## Installation
First of all, you have to install docker. Note that most linux distros comes with docker, but it could be out-dated. So it is best to follow instructions on their [official website](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository). I have chosen to install with repository so that it can be updated easily.

Other than docker, we have to install [nvidia-docker](https://github.com/NVIDIA/nvidia-docker) to utilize GPU.

I prefer using [deepo environment](https://hub.docker.com/r/ufoym/deepo). Note that the documentation on docker hub may be outdated, and github should have the most accurate documentation.

## running container
The main objects in docker is probably images and containers. Containers are instances of images, and thus you can have a single image with multiple containers using the same image. In our case, to run a particular container, run:

```
sudo docker run --gpus all -it -p 0.0.0.0:6006:6006 --rm -v ~/Documents/Docker/deepo/IDS2-CIC_IDS:/deepo/IDS2-CIC_IDS -v /etc/localtime:/etc/localtime:ro {repo_name}/{image_name} bash
```
The options means the following:
- --gpus all tells the container to use all gpus available, this needs nvidia-docker to be installed
- -it opens an interactive terminal
- -p 0.0.0.0:6006:6006 this tells the container to map its ports to localhost. This is done so tensorboard serving at 6006 is mapped to host machine.
- --rm clean up the container after it has exited
- -v maps host folder/files to containers files. In this case, we map the project folder into container, and time folders so that the saved files in container have the correct time
- base states we want to open up a bash shell

## committing and pushing images

Once the container is running, and you install/upgrade packages in apt-get or pip, the changes will not be persistent. Thus you have to run
```
docker commit -m "{message}" {docker_id} {repo/name}
```
You will see that the new image have an extra layer added on top, representing the changes

After the change, you can run docker push {docker_id} to push the changes
