---
layout: post
title:  "Centralized logging for Docker containers"
date:   2016-4-19 14:36:23
category: "Internship week 8"
tags: [docker, logging, fluentd, kibana, elastic]
---

During week 7 & 8 at <a href="www.smalltownheroes.be"><b>Small Town Heroes</b></a>, we researched and deployed a centralized logging system for our Docker environment. We use <a href="www.fluentd.org"><b>Fluentd</b></a> to gather all logs from the other running containers, forward them to a container running <a href="https://www.elastic.co/products/elasticsearch"><b>ElasticSearch</b></a> and display them by using <a href="https://www.elastic.co/products/kibana"><b>Kibana</b></a>. The result is similar to the ELK (Elasticsearch, Logstash, Kibana) stack, only we use Fluentd instead of Logstash.
<!--more-->

## <b> One container to log them all</b>

To manage logging Docker has implemented a system that makes use of <b>Logging Drivers</b>.
By specifying the `--log-driver` in your `docker run` command you container will be configured to use this driver.  

Docker provides these drivers for logging:
<table class="default">
	<thead>
		<tr>
			<th style="text-align: center">Type</th>
			<th style="text-align: center">Description</th>
		</tr>
	</thead>
	<tbody>
    	<tr>
      		<td style="text-align: center"><code class="highlighter-rouge">json-file</code></td>
      		<td style="text-align: center">Default logging driver for Docker. Writes JSON messages to file.</td>
    	</tr>
    	<tr>
      		<td style="text-align: center"><code class="highlighter-rouge">syslog</code></td>
      		<td style="text-align: center">Syslog logging driver for Docker. Writes log messages to syslog.</td>
    	</tr>
    	<tr>
      		<td style="text-align: center"><code class="highlighter-rouge">journald</code></td>
      		<td style="text-align: center">Journald logging driver for Docker. Writes log messages to journald.</td>
    	</tr>
    	<tr>
      		<td style="text-align: center"><code class="highlighter-rouge">gelf</code></td>
      		<td style="text-align: center">Graylog Extended Log Format (GELF) logging driver for Docker. Writes log messages to a GELF endpoint likeGraylog or Logstash.</td>
    	</tr>
    	<tr>
      		<td style="text-align: center"><code class="highlighter-rouge">fluentd</code></td>
      		<td style="text-align: center">Fluentd logging driver for Docker. Writes log messages to fluentd (forward input).</td>
    	</tr>
    	<tr>
      		<td style="text-align: center"><code class="highlighter-rouge">awslogs</code></td>
      		<td style="text-align: center">Amazon CloudWatch Logs logging driver for Docker. Writes log messages to Amazon CloudWatch Logs.</td>
    	</tr>
    	<tr>
      		<td style="text-align: center"><code class="highlighter-rouge">splunk</code></td>
      		<td style="text-align: center">Splunk logging driver for Docker. Writes log messages to splunk using HTTP Event Collector.</td>
    	</tr>
    	<tr>
      		<td style="text-align: center"><code class="highlighter-rouge">etwlogs</code></td>
      		<td style="text-align: center">ETW logging driver for Docker on Windows. Writes log messages as ETW events.</td>
    	</tr>
    	<tr>
      		<td style="text-align: center"><code class="highlighter-rouge">gcplogs</code></td>
      		<td style="text-align: center">Google Cloud Logging driver for Docker. Writes log messages to Google Cloud Logging.</td>
    	</tr>
    </tbody>
</table>

Basically if you would like to see the logs of a specific container, you'd have to use the `docker logs` command. However this command is only compatible with the json-file and journald logging drivers. Naturally this means that for displaying logs via another logging driver, you'd have to have other means of displaying these logs.  
In any case it's clear that there are a lot of options to choose from where most depends on personal choice. 

### <b> The EFK stack</b>

For our solution we went with the default JSON logging driver. This gives us the flexibility to still be able to use the `docker logs` command when we are logged in to the server as well as forwarding our logs to our log storage container. For collecting, storing and displaying the logs we ultimately went with the **elasticsearch-fluentd-kibana or EFK** stack.

#### <b> One container to collect them all </b>

So for collecting all the containers' logs we chose to use **Fluentd**. At first we experimented with using the fluentd logging driver but then a few problems arised. First of all we can not start any containers when the fluentd container is not started due to the required `--log-driver=fluentd`. When executing the `docker run` command, this parameter will tell the container to use the Fluentd logging driver. As a result, the container will fail to start when there is no Fluentd container running.  
Secondly, when the Fluentd container crashes/stops for some reason, all the logs for the period that it wasn't running are lost. At this point the running containers are still sendig their logs to localhost:24224 (the default Fluentd port) yet there isn't anything listening on that port, obviously.  
Lastly, the other great disadvantage that the logging driver system brings, is the fact that all containers' logs are being forwarded as "stdout" to a published port on the fluentd container. We want to be able to filter and format our containers' logs to our own preferences, yet doing this with "stdout" output isn't easy to say the least. The Fluentd logging driver is able to send the following metadata in the log message: `container_id`, `container_name` and `source`. In our case, this simply isn't good enough. Plus, in the case of for example Nginx's logs there are 2 types of log messages: access logs and error logs. Both types use a different logging format. Fluentd log driver will put the whole container's log message in the field `log: ...`. This log message needs still to be parsed to improve readability. Because access and error logs use a different format, you'll have to use your regex magic skills to parse these messages appropriately.  

To break this down, **our reasons for not using the logging driver system:**  
 <ul class="default">
 <li>Using the logging driver system by Docker creates an extra dependency, something we particularly want to avoid.</li>
 <li>In case of failure, there is no option to recover logs. </li>
 <li>Filtering and formatting different types of logs is a serious pain to do when all logs are forwarded as stdout to the same port.</li>
 </ul>


Alternatively, we have our Fluentd container **tail** the logs from the **default JSON-file** logs created by Docker. The obvious benefit of this is the fact that all log messages are neatly structured in a JSON format. Also, Fluentd using a "tail" source type is able to store its current position in a position file to keep track of which log messages he has already tailed and forwarded. When the Fluentd container would stop running, it can just read the position file and continue after it has been restarted. Even while the Fluentd container isn't running, containers' logs are still being saved on the Docker machine. In case of failure, this proves to be an easy back-up solution.

This was the config file we started with.

```
<source>
  type tail
  read_from_head true
  pos_file fluentd-docker.pos
  path /var/lib/docker/containers/*/*-json.log
  time_format %Y-%m-%dT%H:%M:%S
  tag docker.*
  format json
</source>

<match docker.**>
  type elasticsearch
  log_level info
  host elastic
  port 9200
  include_tag_key true 
  logstash_format true
  flush_interval 5s
</match>

```
<br />
With this config file, all that Fluentd does is tail the JSON log files from the Docker directory, store its current position in the file `fluentd-docker.pos` and finally tag them with the prefix "docker". Lastly all the logs with this prefix (so all of them) are sent to the Elasticsearch container. To make the log messages collected by Fluentd compatible with Elasticsearch, we use <a href="https://github.com/uken/fluent-plugin-elasticsearch"><b>fluent-plugin-elasticsearch</b></a>.   
This configuration had almost everything we need except we had no idea what exact container the logs were coming from, the only thing added was the long container id string (which is coming from the filepath of the specific log).  
Specifically, we wanted to have the hostname we originally set when starting the containers. We started using the module <a href="https://github.com/fabric8io/fluent-plugin-docker_metadata_filter"><b>fluent-plugin-docker_metadata_filter</b></a> to provide us with the additional Docker metadata. We ran into a problem with the version on gem repository not being the latest version (and containing a bug) so we had to install it this way.

```
git clone https://github.com/fabric8io/fluent-plugin-docker_metadata_filter && \
      cd fluent-plugin-docker_metadata_filter && \
      gem build fluent-plugin-docker_metadata_filter.gemspec && \
      gem install --no-ri --no-rdoc fluent-plugin-docker_metadata_filter
```
<br />
After installing the ruby gem all we had to do was add this snippet to our fluentd config file:

```
<filter docker.var.lib.docker.containers.*.*.log>
  type docker_metadata
</filter>
```
<br />
Now we have logs with the added docker metadata for that container. The added **Docker metadata** is this: 

<ul class="default">
<li>Container ID</li>
<li>Container name</li>
<li>Container hostname</li>
<li>Image name</li>
<li>Image ID</li>
<li>Labels</li>
</ul>
  
<h4> <b> One container to store them all </b></h4>

As mentioned before, to store all the container's logs we went with an **Elasticsearch** container. Elasticsearch isn't really a regular database but actually a search server based on <a href="https://lucene.apache.org/core/"><b>Apache Lucene</b></a>. Basically Elasticsearch is a "cluster of nodes". A cluster being a collection of one or more servers (a.k.a. nodes) that contains the entire data. The strong point of Elasticsearch which makes it so popular is the fact that it provides a **scalable, nearreal-time search**. Next to that, it's also **massively distributed**. This means that data can be divided into shards (or pieces of data) and each shard can have multiple replicas while each node contains one or more shards. Thus Elasticsearch combines the commonly known practices of **"sharding"** and **"replication"**. If you're not familiar with either of those, this is **why they are important:**


<ul class="default">
<li><b>Sharding</b></li>
  <ul class="default">
    <li>horizontally split/scale your content volume</li>
    <li>distribute and parallelize operations across shards (potentially on multiple nodes) thus increasing performance/throughput</li>
  </ul>
<li><b>Replication</b></li>
  <ul class="default">
    <li>high availability in case a shard/node fails</li>
    <li>scale out your search volume/throughput since searches can be executed on all replicas in parallel</li>
  </ul>
</ul>


#### <b> One container to show them all </b>

Logically, for displaying our containers' logs we chose **Kibana**. Kibana is also a product of Elastic just like Elasticsearch which means the two work together seamlessly. There wasn't really much configuration needed here. Basically we just specified a `--link` with our Elastic container in our `docker run` command for the Kibana container.  
The only thing we added here was means of authentication. There are a lot of complicated ways to do this but we chose to simply add authentication through our nginx. But because we are using docker-gen for this as explained in <a href="/internship%20week%202/2016/02/24/dock-gen-and-continuous-integration.html">our previous blog post</a>, we had to add authentication to our template. This turned out to be rather simple just by adding the next few lines of code inside the 'location /' block

<pre>
<code>
		&#123;{ if hasPrefix "logging-stage" $host }}
		auth_basic	"Restricted &#123;{ $host }}";
		auth_basic_user_file	&#123;{ (printf "/etc/nginx/htpasswd/%s" $host) }};
		&#123;{ end }}
</code>
</pre>
<br />

What this does is checking if the 'VIRTUAL_HOST' variable has a prefix logging-stage and if so, add basic authentication to that location block. The password file is premade and can be copied when building the docker image or just add a volume mounting the file on the desired location.