---
layout: post
title:  "Getting started with docker: Installation"
date:   2016-2-16 14:36:23
category: "Internship week 1"
tags: [docker, fedora]
---

<div style="text-align:center"><img src ="/images/docker_logo.png" style="max-width:100%"/></div>

Before beginning our internship at Small Town Heroes, we didn't have any experience with Docker.  
Since Rome wasn't built in a day, we would learn this technology step-by-step, starting with the basics.  

<!--more-->

## <strong>1. Setting up Docker on Fedora 23</strong>

The first step in this process was to setup Docker locally on our machine.  
In our case this was our laptop with Fedora 23 as OS.  
Basically we needed two parts, the <strong>docker engine</strong> and the <strong>docker machine</strong>.  

## <strong>Docker engine:</strong>

The docker engine is a client-server application consisting of <strong>three elements:</strong>

<div>
<ol class="default">
	<li>The docker daemon</li>
	<li>a REST API</li>
	<li>and a CLI that talks to the daemon through the REST API</li>
</ol>
</div>
<div style="text-align:center"><img src ="/images/engine.png" style="max-width:100%"/></div>


We installed this very easily by using the <strong>dnf package manager</strong>. For all of these steps, it’s important to be logged in with sudo priviliges.  


#### <strong>1. Make sure all of your packages are up-to-date</strong>

```bash
$ sudo dnf update
```
<br />


#### <strong>2. Add the yum repo</strong>

```bash
$ sudo tee /etc/yum.repos.d/docker.repo <<-‘EOF’ 
[dockerrepo] 
name=Docker Repository 
baseurl=https://yum.dockerproject.org/repo/main/fedora/$releasever/ 
enabled=1 
gpgcheck=1 
gpgkey=https://yum.dockerproject.org/gpg 
EOF
```
<br />


#### <strong>3. Install the Docker package</strong>

```bash
$ sudo dnf install docker-engine
```
<br />


#### <strong>4. Start the Docker daemon</strong>

```bash
$ sudo systemctl start docker
```  
<br />


If this works without resulting in any errors, then you’re most likely good to go. But to make sure you can always verify your installation by deploying a test image in a container.


## <strong>Docker machine:</strong>

“Docker Machine is a tool that lets you install Docker Engine on virtual hosts, and manage the hosts with `docker-machine` commands.”

With docker machine, it’s possible to deploy multiple hosts, each running a docker engine, a.k.a. Dockerized hosts or Machines. Each of these hosts can contain multiple containers that are created from an image. The benefit of using docker machine on Linux is to efficiently provision multiple Docker hosts on a network, in the cloud or even locally.

Now back to installing this…

### <strong> 1. Download Docker machine and extract it</strong>

```bash
$ curl -L https://github.com/docker/machine/releases/download/v0.6.0/docker-machine-`uname -s`–`uname -m` > /usr/local/bin/docker-machine && \ chmod +x /usr/local/bin/docker-machine
```
<br />

### <strong> 2. Verify your installation by checking the version</strong>


```bash
$ docker-machine version
docker-machine version 0.6.0, build 61388e9
```

<br />
After this was all done, we were ready to start creating our own docker machine to deploy containers.