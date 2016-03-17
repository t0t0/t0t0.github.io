---
layout: post
title:  "Benchmarking performance & security on Docker containers"
date:   2016-3-18 14:36:23
category: Internship week 4
tags: [docker, performance, security, benchmarking]
---


INTRO

<!--more-->


### Sources

#### Security
https://github.com/docker/docker-bench-security

#### Performance
https://github.com/misterbisson/simple-container-benchmarks

https://github.com/phoronix-test-suite/phoronix-test-suite
## Benchmarking security:  
  
A simple script that can be run locally on the host and that checks for dozens of common best-practices around security for Docker containers in a production environment.  
Obviously, we want all warnings to be passes.  

```
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security
sh docker-bench-security.sh
```
## Benchmarking performance:

### Phoronix test suite

For bechmarking the performance of our Docker containers in our production environment we use "Phoronix test suite", a free open-source benchmarking platform.  

#### 1. Installing Phoronix test suite:

We decided to use a separate container on which the phoronix test suite, including our predefined tests, will be installed. This container's only function would be to deploy benchmarks on our machine.  
As we like to keep our docker images as small as possible, we use <href a="http://www.alpinelinux.org/">alpine</a> as our base image.  
  
The next step is to provide all dependencies of both phoronix test suite as our tests that we will install. Normally phoronix would ask for user input upon installing a test when extra packages need to be installed. This is something we obviously want to avoid while building the docker image.  
After these packages are downloaded and installed, we start with installing the phoronix test suite itself. To do this, we first download the official compressed package and extract it. We then remove the download file so we don't have any unnecessary files on our system. At last we cd into the extracted folder and run the install script.  
  
Lastly, we install our predefined tests and run the command `phoronix-test-suite detailed-system-info`. This command will allow us to always check our system info when checking the logs of this specific container.  
  
All of this results in this Dockerfile below:


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

CMD phoronix-test-suite detailed-system-info
```


#### 2. Running benchmark tests remotely


