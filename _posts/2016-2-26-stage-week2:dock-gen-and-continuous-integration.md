---
layout: post
title:  "Stage week 2: Docker-gen and continuous integration"
date:   2016-2-26 14:36:23
categories: Stage
---
<div style="text-align:center;padding-bottom:25px;"><img src ="../../../../images/circleci_logo.png" style="max-width:100%" /></div>

Our next step in learning docker was setting up continuous integration. For this we used circleci and some other nifty features. This allowed the other interns whom were develloping applications to push their code and it automaticly depolyed.

## 1. Makefiles 

The first thing we used were makefiles. Here we could define our own make commands so we needn't type out the full commands. An example of one of these makefiles can be found below.

```makefile
#####
# vars
####

IMAGE_NAME=demo
IMAGE_VERSION=latest
PORTHOST=80
PORTGUEST=80
DOCKERFILEPATH=./Dockerfile

####
# concatenate vars
####

# full file name for tarbal
TARNAME = $(IMAGE_NAME).tar
# full image name
IMAGE = $(IMAGE_NAME):$(IMAGE_VERSION)

####
# Make commands
####

docker_build:
	docker build -t $(IMAGE) -f $(DOCKERFILEPATH) .

docker_run:
	docker run -d -p  $(PORTHOST):$(PORTGUEST) $(IMAGE); sleep 10

docker_build_run: docker_build docker_run

docker_save:
	docker save -o $(TARNAME) $(IMAGE)
```
<br />

These makefiles also enabled us to nest make commands like `docker_build_run`. This gives us acces to the one command to build and run aswell as both run and build separately.


