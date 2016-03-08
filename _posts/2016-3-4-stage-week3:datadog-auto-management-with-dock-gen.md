---
layout: post
title:  "Stage week 3: datadog auto management with dock-gen"
date:   2016-3-4 14:36:23
categories: Stage
---

<div style="text-align:center"><img src ="../../../../images/datadogtest.png" style="max-width:100%" style="padding-bottom:25px"/></div>

## **Datadog**

In our third week at Small Town Heroes, we started setting up monitoring for our current environment. We use **Datadog** as a monitoring platform.  
Datadog has a neat Docker integration where the Agent is running as an individual container on your Docker-machine.  
This container will gather metrics from the machine and all containers created on this machine and forward them to the Datadog web interface.  
  
Setting this up is as easy as running a single command:

```bash
$ docker run -d --name dd-agent -h `hostname` -v /var/run/docker.sock:/var/run/docker.sock -v /proc/:/host/proc/:ro -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro -v /opt/dd-agent-conf.d:/conf.d:ro -e API_KEY={your_api_key_here} datadog/docker-dd-agent
```
<br />

As we both resent manually executing long commands, we made some make commands for the deployment of our dd-agent container.  

```makefile
datadog_run:
	ssh $(CONNECT) docker run -d --name $(DATADOG_NAME) -h $(HOSTNAME) -v /var/run/docker.sock:/var/run/docker.sock -v /proc/:/host/proc/:ro -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro -v /opt/dd-agent-conf.d:/conf.d:ro -e API_KEY=$(API_KEY) -e SERVICE="datadog" datadog/docker-dd-agent

datadog_cleanup:
	ssh $(CONNECT) docker stop $(DATADOG_NAME) | xargs -r ssh $(CONNECT) docker rm 

datadog_deploy: datadog_cleanup datadog_run	
```
<br />

Basically, this will stop - remove - run our dd-agent container with the parameters that we've specified.  
Why we added the `SERVICE="datadog"` environment variable will be explained later on...  

## **Auto managing Datadog with Docker-gen**
  
#### **Integrations** 

After having the dd-agent container running, it's already possible to monitor and analyze all containers through the web interface.
But aside from the Docker integration that is installed by default on this container, we also wanted to monitor other services such as Nginx, Redis, Apache, etc.  
Now the way Datadog's dd-agent works with integrations is this: within the container there is a YAML file for every integration stored in the config directory `/etc/dd-agent/conf.d/`.  
This means that if you want to add an integration, you'll have to manually edit the specific YAML file for this integration and put it in the config directory on the dd-agent container.  
Since we've already played around with Docker-gen, we figured this could be a lot easier when we have a template for each integration that we can use to let Docker-gen automatically generate the correct YAML file and put it in the right directory on the dd-agent container.  

#### **Tagging**

Another thing we had to think about was **tagging**. Tags can be used by Datadog to filter your results in Dashboards, the Infrastructure list, the Host map and in Monitors.
We decided to use two tags, one to specify the project and one for the environment. In our current setup we are working on two projects that have two possible environments. They're either in staging or in production. These tags are assinged in the config files, thus supplied through the templates that Docker-gen will use.


#### **Docker-gen**



