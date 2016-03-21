---
layout: post
title:  "Benchmarking performance on a Docker container"
date:   2016-3-16 14:36:23
category: Internship week 5
tags: [docker, performance, benchmarking, phoronix]
---


During our fourth week at Small Town Heroes, we were asked to dig deeper into benchmarking both performance and security of docker containers.
This blog post will cover the part about benchmarking performance. We run tests for benchmarking the performance of our disk, cpu, memory and also separate services.

<!--more-->


### **Phoronix test suite**

For bechmarking the performance of our Docker containers in our production environment we use <a href="http://www.phoronix-test-suite.com/">Phoronix test suite</a>, a free open-source benchmarking platform.  

#### **1. Building the docker-phoronix image**

We decided to use a separate container on which the phoronix test suite, including our predefined tests, will be installed. This container's only function would be to deploy benchmarks on our machine.  
As we like to keep our docker images as small as possible, we use <a href="http://www.alpinelinux.org/">alpine linux</a> as our base image.  
  
The next step is to provide all dependencies of both phoronix test suite as our tests that we will install. Normally phoronix would ask for user input upon installing a test when extra packages need to be installed. This is something we obviously want to avoid while building the docker image.  
After these packages are downloaded and installed, we start with installing the phoronix test suite itself. To do this, we first download the official compressed package and extract it. We then remove the downloaded file so we don't have any unnecessary files on our system. Then we 'cd' into the extracted folder and run the install script.  
In our case, we install some predefined tests by running the command `phoronix-test-suite install [test]`    

Lastly, our custom scripts are copied onto the container and we set our main script `run.sh` to be executed on run.  

This results in the Dockerfile below:


```
#####################
# Bench Dockerfile #
#####################

# Set the base image 
FROM    alpine

# File Author / Maintainer
MAINTAINER Toon Lamberigts and Tomas Vercautter

# Install dependencies
RUN apk update && apk add --no-cache make gcc g++ libtool linux-headers perl pcre-dev php php-dom php-zip php-json

# Download  & extract Phoronix package
RUN wget http://www.phoronix-test-suite.com/download.php?file=phoronix-test-suite-6.2.2 -O phoronix-test-suite-6.2.2.tar.gz
RUN tar xzf phoronix-test-suite-6.2.2.tar.gz
RUN rm -f phoronix-test-suite-6.2.2.tar.gz
RUN cd phoronix-test-suite && ./install-sh

# Install tests
## Disk
RUN phoronix-test-suite install pts/iozone
## CPU
RUN phoronix-test-suite install pts/c-ray
## Memory
RUN phoronix-test-suite install pts/stream
## Services
RUN phoronix-test-suite install pts/apache
RUN phoronix-test-suite install pts/redis


# Copy custom scripts
COPY scripts/ .

# Execute benchmark script
CMD ./run.sh
```
<br />

#### **2. Deploying the docker-phoronix container**

To deploy a container with this image on our remote server, we created **a Makefile workflow** for our own ease of use.
First we built the docker image locally and uploaded it to our remote server afterwards. But because of a slow internet connection, uploading the 295MB image took a little too long for our liking.  
Therefore we decided to only copy our scripts and Dockerfile from our local directory to the remote server and build the image there.  


```Makefile
bench_build:
	scp -r $(LOCAL_DIR)/* $(CONNECT):$(REMOTE_DIR)
	ssh $(CONNECT) docker build -t docker-phoronix -f $(REMOTE_DIR)/Dockerfile $(REMOTE_DIR)

```
<br />
The deployment of our docker image is as easy as running the command `make bench_build`  

#### **3. Usage**

Now to actually run our benchmark tests on the CoreOS remote server, we created a few more make commands.  


```Makefile
bench_run:
	@ssh $(CONNECT) docker run -i --name docker-phoronix docker-phoronix 

bench_cleanup_container:
	@echo "Removing container..."
	@ssh $(CONNECT) docker stop docker-phoronix | xargs -r ssh $(CONNECT) docker rm 

bench: bench_run bench_cleanup_container
```
<br />

Upon executing the command `make bench`, our docker-phoronix container is ran which results in our custom script to be executed. This script returns a menu where you can choose one of the options that we implemented. After exiting the menu, the docker-phoronix container will be removed automatically.

<div style="text-align:center;padding-bottom:25px;"><img src ="/images/docker-phoronix.png" style="max-width:100%" /></div>

To try this project yourself, head over to our <a href="https://github.com/t0t0/docker-phoronix">Github repository!</a>