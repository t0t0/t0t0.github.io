---
layout: post
title:  "Benchmarking performance & security on Docker containers"
date:   2016-3-18 14:36:23
category: Internship
tags: [week 4, docker, performance, security, benchmarking]
---


INTRO

<!--more-->

## Benchmarking performance & security on Docker containers

### 1. Sources

#### Security
https://github.com/docker/docker-bench-security

#### Performance
https://github.com/misterbisson/simple-container-benchmarks

https://github.com/phoronix-test-suite/phoronix-test-suite


### 2. Performance benchmarking without any installed software:

### 3. Performance benchmarking with installed software:

#### Sysbench

On our apache containers: `apt-get update && apt-get install -y sysbench`


#### Phoronix

On our node containers there is no package available for sysbench so we turned to "Phoronix test suite".

We will run this software locally from the extrated tar.gz package. The only dependency to use phoronix test suite is having CL support for PHP installed.

1. ** Installing command-line support for PHP**

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





