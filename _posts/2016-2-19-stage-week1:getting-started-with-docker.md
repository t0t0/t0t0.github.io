---
layout: post
title:  "Internship week 1: Getting started with docker"
date:   2016-2-19 14:36:23
categories: [Internship, Docker]
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



## <strong>2. Deploying Docker containers locally</strong>


To start getting familiar with Docker, we started out with creating a local environment consisting of two NodeJS containers, a Redis container as a Database and an Nginx container to be used as a load-balancer. 
We based ourselves on <a href="http://anandmanisankar.com/posts/docker-container-nginx-node-redis-example/">this sample workflow</a>.  


<div style="text-align:center"><img src="/images/DockerSample.png" alt="Sample Image by ANAND MANI SANKAR" style="max-width:100%"/></div> <br />

As we started out, we deployed our containers by manually issueing the `docker-run` command. 
Because managing multiple containers is rather inefficient with this command, we decided to work with `docker-compose`, a tool for defining and running multi-container Docker applications.  

Very much like working with Ansible & Vagrant, which we are both experienced by, `docker-compose` automatically deploys (in this case container instead of VM's) based on specifications that are defined in a YML-file.   
<br />
Our `docker-compose.yml` looked like this:  

```yml
nginx:
    container_name: nginx
    build: ./nginx
    links:
        - node1:node1
        - node2:node2
    ports:
        - "8000:80"
node1:
    container_name: node1
    build: ./node
    links:
        - redis
    expose:
        - "3000"
node2:
    container_name: node2
    build: ./node
    links:
        - redis
    expose:
        - "3000"
redis:
    container_name: redis
    image: redis
    ports:
        - "6379"
```
<br />
As you can see, we used our own Dockerfiles for both the Nginx and NodeJS containers. Those were located in the folder 'ngninx' and 'node' respectively.  
To orchestrate the relationships between containers we made use of the `--link` property. This enables a specific container to resolve the addresses of other containers by name or alias.  
The only port that is published by the docker-machine to allow access from the outside is port 8000 which is mapped to port 80 on the Nginx container.  
Running the command `docker-compose up` would automatically deploy our containers in order of their dependency.  


## <strong> 3. Deploying containers on a remote server</strong>

The next step was to deploy the Docker containers remotely. This time we would run a NodeJS application, developed by our colleague, on both node containers.  
This remote server was a **CoreOS VM running on AWS**.

<div style="text-align:center"><img src="/images/coreos.png" style="max-width:100%"/></div> <br />
 
We immediately ran into a major problem: CoreOS doesn't have a Python interpreter installed which is necessary to execute Ansible commands, nor does it have a package manager to install Python.  
Luckily, Ansible has a raw execution mode which allows shell-commands to be executed directly on the remote machine. Via this method in combination with an existing <a href="https://github.com/defunctzombie/ansible-coreos-bootstrap">coreos-bootstrap Ansible role</a> we were able to get Ansible running after all.  
To read more about this solution, check out <a href="https://coreos.com/blog/managing-coreos-with-ansible/">this blog post</a>.  

We created our own ansible role which would be executed for our CoreOS host via SSH.  
Within this role we firstly create the two node containers, generate a custom Nginx config file based on our template and finally create the Nginx container where this config file is mounted on.  

Our Ansible tasks looked like this:

```
- name: Install docker-py as a workaround for Ansible issue
  pip: name=docker-py version=1.2.3
- name: Setup node containers
  docker:
   count: "&#123;{ node_count }}"
   image: dev-node
   state: "&#123;{ node_state }}"
   expose:
   - 3000
- name: Generate nginx config file
  template:
    src: nginx.conf.j2
    dest: /home/core/config/nginx/nginx.conf
- name: Setup Nginx container
  docker: 
   name : nginx
   image: dev-nginx
   state: "&#123;{ nginx_state }}"
   ports:
   - "80:80"
   volumes:
   - /home/core/config/nginx/nginx.conf:/etc/nginx/nginx.conf
   links:
   - "node1:node1"
   - "node2:node2"
```

<br />

For both parameters 'count' and 'state' we declared variables. The images 'dev-node' and 'dev-nginx' were our own built images according to our Dockerfiles.




