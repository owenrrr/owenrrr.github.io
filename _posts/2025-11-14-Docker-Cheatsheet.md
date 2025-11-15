---
layout: post
title:  "Docker Cheatsheet"
date:   14 Nov 2025
categories: Demo
tags: Tools
---
<html>
<body>
<div markdown="block" style="margin-top: 10px">
    
### Intro
Docker CLi Cheatsheet that could follow when I forget about the basic commands :)

### Docker Pull

- docker build image by Dockerfile

```bash
#docker build -t <image_name>:<tag> .
docker build -t my_image:latest .
```

- docker pull image from remote repo

```bash
#docker pull <image_name>:<tag>
docker pull ubuntu:22.04
```

### Docker Image

```bash
# show docker images
docker images
# delete image by <image_id>
docker rmi IMAGE_ID
# delete image by <image_name>:<tag>
docker rmi ubuntu:22.04
```

### Docker Container

```bash
# docker build container via image & run it via interactive mode
docker run -it --name my-ubuntu ubuntu:22.04 bash
# docker run on daemon mode, host:container port matching
docker run -d --name my-nginx -p 8080:80 nginx
# show all running docker containers
docker ps
# show all docker containers
docker ps -a
# docker start/stop/restart container <container_name>
docker start my-ubuntu
docker stop my-ubuntu
docker restart my-ubuntu
# delete container by <container_name>
docker rm my-ubuntu
# have an interactive TTY while you have a running container
docker exec -it my-ubuntu bash
```

### Docker File Operations

- Copy file from local to container

```bash
docker cp tester.cc my-ubuntu:/home/tester.cc
```

- Copy file from container to local

```bash
docker cp my-ubuntu:/home/tester.cc ./tester.cc 
```

## Materials
- [dockerdocs](https://docs.docker.com/reference/cli/docker/)

</div>
</body>
</html>
