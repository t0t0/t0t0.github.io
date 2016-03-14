---
layout: post
title:  "Docker-gen & Nginx as a proxy web-server"
date:   2016-2-26 14:36:23
category: "Internship week 2"
tags: [nginx, docker-gen]
---


When we started using CircleCI for auto deployment a problem arised. Every time code got pushed and deployed to our CoreOS server, new containers with new ip-adresses were created. Because of this we had to go into the nginx container to adjust the configuration. This wasn't very continuous so we had to find a solution for this.

<!--more-->

<div style="text-align:center"><img src ="/images/stageWeek2/flow.png" style="max-width:100%"/></div>

This is where we started using docker-gen. Docker-gen is a file generator that, using a template en docker meta-data, can generate files. It can listen on the docker socket if any containers are stopped or started and regenerate the files with the new data. More information on docker-gen can be found <a href="https://github.com/jwilder/docker-gen">here</a>

For our nginx problem we used an implementation of docker-gen made by jason wilder on github called nginx-proxy. 
The easy way to set this up is by running the next command.

```bash
docker run -d -p 80:80 -v /var/run/docker.sock:/tmp/docker.sock:ro jwilder/nginx-proxy
```
<br />
This wil set up an nginx container with the docker-gen embedded. The only thing you need to do is adapt your docker run command for the other containers so that it includes `-e VIRTUAL_HOST=subdomain.youdomain.com` where you replace the `subdomain.youdomain.com` with your domain information.

We already had a customized nginx container and we didn't want to reconfigure that one so we opted for the other way of setting this up, using a separate container to run docker-gen.

The nginx container now required `-v /tmp/nginx:/etc/nginx/conf.d` in its run command and the docker-gen container was run with:

```bash
 docker run --volumes-from nginx \
    -v /var/run/docker.sock:/tmp/docker.sock:ro \
    -v $(pwd):/etc/docker-gen/templates \
    -t jwilder/docker-gen \
    -notify-sighup nginx -watch -only-exposed /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
```
<br />
For more information on docker-gen and nginx-proxy you can use the following links

<div>
<ul class="default">
	<li><a href="https://github.com/jwilder/nginx-proxy"> nginx-proxy github page</a></li>
	<li><a href="http://jasonwilder.com/blog/2014/03/25/automated-nginx-reverse-proxy-for-docker/">nginx-proxy blogpost by jason wilder</a></li>
	<li><a href="https://github.com/jwilder/docker-gen"> docker-gen github page</a></li>
</ul>
</div>
