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
Obviousky, we want all warnings to be passes.  

```
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security
sh docker-bench-security.sh
```
## Benchmarking performance:

### Phoronix test suite

On our nodeJS containers (running on Alpine Linux) there is no packages available like for example "sysbench" so we turned to "Phoronix test suite".

We will run this software locally from the extracted tar.gz package. The only dependency to use phoronix test suite is having CL support for PHP installed.  
Note that specific test could require extra packages to be installed.

1. **Installing command-line support for PHP**

```shell
$ apk update && apk add php php-dom php-zip php-json
```
2. **Downloading and extracting the Phoronix test suite package**

```shell
$ wget http://www.phoronix-test-suite.com/download.php?file=phoronix-test-suite-6.2.2 -O phoronix-test-suite-6.2.2.tar.gz
$ tar xzf phoronix-test-suite-6.2.2.tar.gz
```

3. **Installing Phoronix test suite**  
In the extracted directory, execute this command:  
```shell
$ ./install-sh
```
#### Installing tests:

Via http://openbenchmarking.org/

```shell
$ phoronix-test-suite install [TEST NAME]
```

#### Filesystem benchmarking: 

```shell
$ phoronix-test-suite benchmark pts/iozone
```

#### CPU benchmarking:


```shell
$ phoronix-test-suite benchmark pts/c-ray
```  
  
Test system's audio/video encoding performance using ffmpeg.  

```shell
$ phoronix-test-suite benchmark pts/ffmpeg
```

#### Memory benchmarking: 

```shell
$ phoronix-test-suite benchmark pts/stream
```

### Service specific benchmarks:

#### Apache:

```shell
$ phoronix-test-suite benchmark pts/apache
```

#### Nginx:

```shell
$ phoronix-test-suite benchmark pts/nginx
```
#### Redis:

```shell
$ phoronix-test-suite benchmark pts/redis
```





