---
layout: post
title:  "Implementing server inaccesability when booting for nginx with docker-gen"
date:   2016-3-13 15:22:23
category: "Internship week 4"
tags: [nginx, docker-gen, CI]
---

This week we got a request, what if the app on your container takes a while to start and it isn't allowed to be acessed yet.
In a development environment it doesn't really matter if your applications take 2-3 minutes to load and the webpage isn't acessible.
But when you're patching live environments of proxy forwards to a server that isn't accessible for 2-3 minutes, thats quite a while.

<!--more-->

## **Adapting nginx template file for config**

The only way I've found to change running containers is to update their name. So the solution I went with was checking if the Container we didn't want requests to get forwarded to has a prefix "startup". If it dit have the prefix "startup" I used the existing code in the template to
add the host to the upstream list but setting it to down in the config. 

<pre>
<code>
&#123;{ range $host, $containers := groupByMulti $ "Env.VIRTUAL_HOST" "," }}
upstream &#123;{ $host }} {
&#123;{ range $container := $containers }}
&#123;{ $name := $container.Name }}
&#123;{ if hasPrefix "startup" $name }}
		&#123;{ template "upstream" (dict "Container" $container) }}
&#123;{else}}
	/* original code */
&#123;{ end }}
&#123;{ end }}
}
&#123;{ end }}
</code>
</pre>

By not adding an "Adress" in the line <code>&#123;{ template "upstream" (dict "Container" $container) }}</code> the template upstream generated a line with the ip and set it to down. This way when the Container is fully started we just remove the "down" part and the server is acessible.

But then we had a problem. Apparently a rename on a container did not trigger an update for docker-gen to regenerate the config files. So now we had to find a way to manually trigger an update for docker-gen after renaming our started container. For this we used the `docker kill` command. `docker kill` allows us to send signals to containers. The default signal is sends is the kill signal, hence the name `docker kill`. With this `kill` command we send a `sighup` do our docker-gen and it regenerates our config files for nginx.

## **Usage** 

### 1. Start container

First we need to start our container using `startup` as a prefix. This we do with the following command:

```bash
docker run -d --name startup$(your container name) --expose $(exposed port) $(container image)
```

The exposed port here isn't required if you already defined an exposed port in the Dockerfile. It just requires as docker-gen only watches containers with exposed ports. As for the rest of the command, it can be modified to fit your implementation. The only required part is the Prefix defined in Go template from before.

### 2. Check if application is started

The next thing we need to do is checking if the application that is running on our container is started. This is something that will differ from project to project but is most easy done with a simple bash script that loops untill it finds that the application is started.

### 3. Rename our container and send signal to docker-gen

When our application is started we need to rename the container so it no longer has startup as it prefix and send the docker-gen a signal to update the config files and signal nginx to reload its config. This is done with:

```bash
docker rename startup$(your container name) $(your conainter name)

docker kill -s HUP $(docker-gen container name)
```

After executing this the container is no longer set to down in the nginx config file and is accessible.


