---
layout: post
title:  "Benchmarking performance on Docker containers"
date:   2016-3-16 14:36:23
category: Internship week 5
tags: [docker, performance, benchmarking, phoronix]
---


During our fourth week at Small Town Heroes, we were asked to dig deeper into benchmarking both performance and security of docker containers.
This blog post will cover the part about benchmarking performance. We run tests for benchmarking the performance of our disk, cpu, memory and also separate services.

<!--more-->


### **Phoronix test suite**

For bechmarking the performance of our Docker containers in our production environment we use "Phoronix test suite", a free open-source benchmarking platform.  

#### **1. Installing Phoronix test suite**

We decided to use a separate container on which the phoronix test suite, including our predefined tests, will be installed. This container's only function would be to deploy benchmarks on our machine.  
As we like to keep our docker images as small as possible, we use <a href="http://www.alpinelinux.org/">alpine linux</a> as our base image.  
  
The next step is to provide all dependencies of both phoronix test suite as our tests that we will install. Normally phoronix would ask for user input upon installing a test when extra packages need to be installed. This is something we obviously want to avoid while building the docker image.  
After these packages are downloaded and installed, we start with installing the phoronix test suite itself. To do this, we first download the official compressed package and extract it. We then remove the downloaded file so we don't have any unnecessary files on our system. Then we 'cd' into the extracted folder and run the install script.  
Lastly, we install our predefined tests by running the command `phoronix-test-suite install [test]`    

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

```
<br />

#### **2. Deploying the docker-bench container**

To deploy a container with this image on our remote server, we created **a Makefile workflow**.  
Firstly we build our docker image, save it to a tarball and upload it to our server. Then we load this image from the tarball.

```Makefile
bench_build:
	docker build -t docker-bench -f $(DOCKERFILE_LOCATION) .

bench_save:
	docker save -o docker-bench.tar docker-bench

bench_upload:
	scp docker-bench.tar $(CORE_IP):$(CORE_LOCATION)

bench_load: 
	ssh $(CONNECT) docker load -i docker-bench.tar
```
<br />
After our docker image has been put on the CoreOS remote server, we can deploy our container.  
We stop and remove a container that is possibly running an older image and remove that older image (that has been renamed after loading the new image but still exists) as well.  

```Makefile
bench_run:
	ssh $(CONNECT) docker run -dit --name docker-bench docker-bench 

bench_cleanup:
	ssh $(CONNECT) docker stop docker-bench | xargs -r ssh $(CONNECT) docker rm 
	ssh $(CONNECT) docker images -q --filter "dangling=true" | xargs -r ssh $(CONNECT) docker rmi
```
<b />

To make this deployment automatic, we apply this workflow:

```Makefile
bench_deploy_container: bench_cleanup bench_run

bench_deploy_image: bench_build bench_save bench_upload bench_load 

bench_deploy: bench_deploy_image bench_deploy_container
```
<br />

#### **3. Running benchmark tests remotely**

Now to actually run our benchmark tests on the CoreOS remote server, we created a couple of make commands.  
Each benchmark test has its own command starting with the prefix 'bench_' followed by their name.

In our example, this looks like this:

```Makefile

bench_disk:
	@ssh $(CONNECT) docker exec -i docker-bench phoronix-test-suite run pts/iozone
	@exit

bench_cpu:
	@ssh $(CONNECT) docker exec -i docker-bench phoronix-test-suite run pts/c-ray
	@exit

bench_redis:
	@ssh $(CONNECT) docker exec -i docker-bench phoronix-test-suite run pts/redis
	@exit

bench_apache:
	@ssh $(CONNECT) docker exec -i docker-bench phoronix-test-suite run pts/apache
	@exit

bench_memory:
	@ssh $(CONNECT) docker exec -i docker-bench phoronix-test-suite run pts/stream
	@exit
```
<br />

For example, running the command `make bench_memory` locally will execute the benchmark test "stream" on our docker container through an SSH connection and display the results in our terminal on the local machine. Pressing "ENTER" will close the connection after the bechmark is fininshed.

<div style="text-align:center;padding-bottom:25px;"><img src ="/images/bench_memory.png" style="max-width:100%" /></div>