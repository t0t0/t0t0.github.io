---
layout: post
title:  "Internship week 3: datadog auto management with dock-gen"
date:   2016-3-4 14:36:23
categories: [Internship, Datadog, Docker-gen]
---

<div style="text-align:center"><img src ="../../../../images/datadogtest.png" style="max-width:100%;padding-bottom:25px"/></div>

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

The use of Docker-gen allows us to automatically generate a config file whenever an event (like killing or creating a new container) occurs.  
We'll configure a dock-gen container specifically for use with Datadog's dd-agent.  


The configuration for this dock-gen container looks like this:

```

[[config]]
template = "/etc/docker-gen/templates/docker_daemon.yaml.tmpl"
dest = "/etc/dd-agent/conf.d/docker_daemon.yaml"
watch = true
[config.NotifyContainers]
dd-agent = 1  # 1 is a signal number to be sent; here SIGINT


[[config]]
template = "/etc/docker-gen/templates/nginx.yaml.tmpl"
dest = "/etc/dd-agent/conf.d/nginx.yaml"
watch = true
[config.NotifyContainers]
dd-agent = 1  # 1 is a signal number to be sent; here SIGINT


[[config]]
template = "/etc/docker-gen/templates/redisdb.yaml.tmpl"
dest = "/etc/dd-agent/conf.d/redisdb.yaml"
watch = true
[config.NotifyContainers]
dd-agent = 1  # 1 is a signal number to be sent; here SIGINT


[[config]]
template = "/etc/docker-gen/templates/apache.yaml.tmpl"
dest = "/etc/dd-agent/conf.d/apache.yaml"
watch = true
[config.NotifyContainers]
dd-agent = 1  # 1 is a signal number to be sent; here SIGINT
```
<br />

Each integration has its own block containing the source template file, the destination where dock-gen should put the generated config file and a notifier to notify the dd-agent container after a new config file is generated.

#### **Nginx Integration**

To enable the Nginx integration for Datadog, we needed more than just a YAML file. It's also necessary to enable the Nginx status page.  
This requires Nginx to be compiled with the <a href="http://nginx.org/en/docs/http/ngx_http_stub_status_module.html">HttpdStubStatusModule</a>, which was turned on by default in our case.  

To check whether or not this is the case, execute the following command:

```bash
$ nginx -V 2>&1 | grep -o with-http_stub_status_module
```
<br />  
You should get `with-http_stub_status_module` as output.  
  
Next, the Nginx configuration file needs to be edited. The following code should be added inside a `server{...}` block.

```
	location /nginx_status {
	  stub_status on;
	  access_log   off;
	  allow x.x.x.x;
	  deny all;
	}
```
<br />

Obviously `x.x.x.x` has to be replaced by either the IP address or CIDR of the machines that are supposed to access this page.  
For this instance, that would be the Datadog container.  
  
Because the IP address of that particular container isn't static and we aren't a fan of hardcoding IP adresses into config files, we had to find a more efficient and dynamic way to add this bit of code to our config file.  

As a solution, we created a small template config file specifically for using the Datadog Nginx integration:

<pre>
<code>
&#123;{ range $host, $value := .}}
&#123;{ $service := coalesce $value.Env.SERVICE ""}}
&#123;{ if eq $service "datadog"}}
server {
	listen 80;

	location /nginx_status {
          stub_status on;
          access_log off;
 		  allow &#123;{ $value.IP }};
          deny all;
        }
}
&#123;&#123; end }}
&#123;{ end }}
</code>
</pre>
<br />

Now to come back to why we supplied our `docker run` command with the `SERVICE` environment variable... We use this variable in all of our templates to distinguish the container(s) we need as you can see in the example above. 

#### **Apache Integration**


Similar to how we set up the Nginx integration, Apache requires a status page to be enabled.  
To do this, we added a bit of code to the Apache config file.

```
<Location "/server-status">
    SetHandler server-status
    Require host dd-agent
</Location>
```
<br />

In contrast to Nginx, it's possible to set a hostname in the `Require` statement. This solves the problem of dd-agent not having a static IP address as mentioned before.  
The only thing to make this work, is to configure a `--link` between both the dd-agent container and the apache container. This will allow the Apache container to resolve the dd-agent's hostname to its IP address.  

## **The result**

To check out all we've just mentioned for yourself, head over to our <a href="https://github.com/t0t0/DataDog_Config_Generator">GitHub repository</a>.  
Feel free to comment on and contribute to our project!

