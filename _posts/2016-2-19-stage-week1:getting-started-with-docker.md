---
layout: post
title:  "Stage week 1: Getting started with docker"
date:   2016-2-19 14:36:23
categories: Stage
---

<div style="text-align:center"><img src ="../../../../images/docker_logo.png" /></div>

Before beginning our internship at Small Town Heroes, we didn't have any experience with Docker.  
Since Rome wasn't built in a day, we would learn this technology step-by-step, starting with the basics.  



## <strong>1. Setting up Docker on Fedora 23</strong>

The first step in this process was to setup Docker locally on our machine.  
In our case this was our laptop with Fedora 23 as OS.  
Basically we needed two parts, the <strong>docker engine</strong> and the <strong>docker machine</strong>.  

### <strong>Docker engine:</strong>

The docker engine is a client-server application consisting of <strong>three elements:</strong>

<div>
<ol class="default">
	<li>The docker daemon</li>
	<li>a REST API</li>
	<li>and a CLI that talks to the daemon through the REST API</li>
</ol>
</div>
<div style="text-align:center"><img src ="../../../../images/engine.png" /></div>


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