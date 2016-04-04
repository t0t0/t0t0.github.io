---
layout: post
title:  "Configuring cloud-config with systemd for a Docker environment"
date:   2016-3-30 14:36:23
category: "Internship week 7"
tags: [coreos, docker, systemd]
---


<div style="text-align:center"><img src ="/images/core_and_docker.png" style="max-width:100%;padding-bottom:25px"/></div>

This week we researched and implemented a way to deploy all of our infrastructure containers automatically on booting or rebooting our CoreOS machine. By defining systemd units in our cloud-config document that will be supplied to our machine through user-data, we created an automatic and continuous Docker environment. 
<!--more-->

## **CoreOS Cloud-init**

CoreOS cloud-init allows a user to customize their CoreOS machines by providing either a `cloud-config` yaml file or an executable script through user-data. You're able to configure certain machine parameters, launch systemd units, etc on boot. 
As said this can be done through user-data which, in our case, would be via the Amazon web console where you can copy-paste the contents of your cloud-config file into the user-data input box. Any changes made to the cloud-config will only be in effect after the machine is stopped or recreated. 

<div style="text-align:center"><img src ="/images/ec2-instance-cloud-config.png" style="max-width:100%;padding-bottom:25px"/></div>
.
To check whether your cloud-config doesn't contain any syntax errors and is valid for use, CoreOS provided a handy tool called <a href="https://coreos.com/validate/">Cloud-Config Validator</a>.
Or you could perform this check by using the coreos-cloudinit binary and providing the `-validate` flag.

## **Running Docker containers with Systemd**

Now we arrive to how we want to use this cloud-init. Upon startup we want to deploy our entire infrastructure.  
At this moment **our infrastructure** consists of these Docker containers:  

<div style="text-align:center"><img src ="/images/infra.png" style="max-width:100%;padding-bottom:25px"/></div>

This means that **9 containers** need to be up and running after inital boot and on every reboot. The way we do this is by running each container as a **systemd unit**.   


### **Systemd & units**

Systemd is an init system used to manage processes and resources on your system. A unit in systemd is a service, a socket, a device, a mount point, etc that can be managed by your system. These units are defined and configured within **unit files**. Through our CoreOS cloud-config, we specify these systemd units to be written and started on boot.  

An example of a unit definition in our `cloud-config.yml`:

```
 - name: "docker-elastic.service"
      command: "start"
      content: |
        [Unit]
        Description=Run an elastic container
        Author=Toon Lamberigts & Tomas Vercautter
        Requires=docker.service
        After=docker.service

        [Service]
        Restart=always
        ExecStartPre=-/usr/bin/docker rm elastic
        ExecStart=/usr/bin/docker run --rm --name elastic elasticsearch
        ExecStop=/usr/bin/docker stop -t 2 elastic
```
<br />
In this example we created a systemd unit for our elasticsearch container which will define a new service `docker-elastic.service` that will be started after the Docker daemon service has started. This container is obviously dependent on the Docker service, hence `Requires` and `After`. Before the container is started, a possibly existing elastic container is removed. This is particulary necessary because on reboot all exisiting containers will only be stopped and not removed. This is defined in the `ExecStartPre` statement. On initial boot there won't be any containers to be removed which results in this command returning a non zero exit status. By putting a '-' in front of the command, the service won't fail even if the command would fail.  
Notice that all commands must have an absolute execution path.

## **Provisioning our CoreOS server**

Most of our containers require custom configuration files, templates and/or Docker image. To supply these upon startup of our CoreOS machine we decided to compress these into a tarball that is uploaded on Amazon S3. We created a service called `provision.service` that will download and extract this tarball with all necessary files from S3. As we implement continuous integration into our deployment, we use **CircleCi** to pack our files and build our images. See <a href="/internship%20week%202/2016/02/24/dock-gen-and-continuous-integration.html">our previous blog post</a> to learn more about how we make use of CircleCi. 

In our `circle.yml` file we will run these tasks in this order:

<ol class="default">
	<b><li>Dependencies</li></b>
	<ul class="default">
 		<li>Build Nginx Docker image</li>
 		<li>Build Fluent Docker image</li>
 	</ul>
 	<b><li>Test</li></b>
 	<ul class="default">
 		<li>Run Elasticsearch container</li>
 		<li>Run Fluent container</li>
 		<li>Test if Fluent container is running</li>
 		<li>Display Fluent logs (purpose of troubleshooting in case of failed test)</li>
 		<li>Run Nginx container</li>
 		<li>Test curling to Nginx webpage</li>
 	</ul>
 	<b><li>Deployment</li></b>
 	<ul class="default">
 		<li>Save Fleunt image to tar</li>
 		<li>Save Nginx image to tar</li>
 		<li>Pack files</li>
 		<li>Pack files + compressed images</li>
 		<li>Upload to S3</li>
 	</ul>
</ol>


A few things must be noted here:  
To test our Fluent container we issue a rather simple if statement

```
fluent_test:
	if ! $$(docker inspect -f {{.State.Running}} fluentd); then exit 2; fi;
```
<br />

This will return an exit status=2 if the container's state isn't equal to running. To have some basic troubleshooting without having to SSH into CircleCi's VM, we also make the container return its logs.  

We save our built images to tarballs and compress it together with our custom config files/templates and upload it all together to our S3 storage in the final step. 
  
In our `cloud-config` we will write a systemd unit to download this tarball from S3 and extract its content into the home directory.

## **Running Docker containers automatically**

Finally we come to the good part, making all our Docker containers run automatically on startup. As said before, every Docker container will be ran as a service that is defined in a systemd unit. Just like in the example above, the preset stays the same:  

Firstly we remove any existing containers that can be existing after a reboot.  Secondly, depending on the fact if the container is using our custom image or not, we load that image from a tarball.  
Lastly we run the container by using the `docker run` command and provide any necessary variables/flags.  

### **Implementing dependencies into systemd**
When a container is dependant of another container like in the case of a `--link` flag, we'll have to make sure that the services to which these containers are attached, are being started in the correct order. For example, our Fluent and Kibana containers can't run before the Elastic container is created because they are both linked to this container. To implement this dependency into systemd, you can use the parameter `After` and `Requires`.  
  
**Requires:** If this unit gets activated, the units listed here will be activated as well. If one of the other units gets deactivated or its activation fails, this unit will be deactivated.

**After:** If a unit foo.service contains a setting After=bar.service and both units are being started, foo.service's start-up is delayed until bar.service is started up.
  
Going by this definition, our `docker-fluentd.service` needs a "Requires" statement as well as an "After" statement for `docker-elastic.service`.

This results in this unit definition:

```
    - name: "docker-fluentd.service"
      command: "start"
      content: |
        [Unit]
        Description=Run a fluentd container
        Author=Toon Lamberigts & Tomas Vercautter
        Requires=docker-elastic.service
        After=provision.service
        After=docker-elastic.service

        [Service]
        Restart=always
        ExecStartPre=/usr/bin/docker load -i /home/core/fluent.tar
        ExecStartPre=-/usr/bin/docker rm fluentd
        ExecStart=/usr/bin/docker run --rm -p 24224:24224 --name fluentd --link elastic -v /home/core/log/fluent.conf:/fluentd/etc/fluent.conf -v /var/lib/docker/containers:/var/lib/docker/containers fluent:latest
        ExecStop=/usr/bin/docker stop -t 2 fluentd
```
<br />
As you can see, we make use of our own Docker image that will be loaded from a tarball. Also, our own custom config file will be mounted on this container on run.  
  
#### **Cleaning up older images**
A final issue we encountered is when loading a Docker image from a tarball that already exists. Instead of overwriting, the older one gets renamed to "<none>" and is actually kept. To remove these old Docker images we wrote a very simple cleanup script which will be run after the fluent or nginx image has been loaded. This script is written on boot itself so we don't have to supply it remotely. To do this we utilize the <code class="highlighter-rouge">write_files</code> section in our <code class="highlighter-rouge">cloud-config</code>.

```
write_files:
  - path: "/home/core/ImageCleanup.sh"
    content: |
       #! /bin/bash
       docker images -q --filter "dangling=true" | xargs -r docker rmi
```
<br />

If everything mentioned here (provisioning, CI, cloud-config file) is configured correctly, then you should see all of your containers being deployed automatically when booting or rebooting your CoreOS machine.