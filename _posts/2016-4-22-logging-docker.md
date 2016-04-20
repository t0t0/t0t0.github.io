---
layout: post
title:  "Centralized logging for Docker containers"
date:   2016-4-22 14:36:23
category: "Internship week 8"
tags: [docker, logging, fluentd, kibana, elastic]
---

During week 7 & 8 at <a href="www.smalltownheroes.be">Small Town Heroes</a>, we researched and deployed a centralized logging system for our Docker environment. We use <a href="www.fluentd.org">Fluentd</a> to gather all logs from the other running containers, forward them to a container running <a href="https://www.elastic.co/products/elasticsearch">ElasticSearch</a> and display them by using <a href="https://www.elastic.co/products/kibana">Kibana</a>.
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

### <b> The stack</b>

For our solution we went with the default json logging driver. This gives us the flexibility to use the `docker logs` command when we are logged in to the server as well as forwarding our logs to our log storage container. For collecting, storing and displaying the logs we went with the fluentd-elasticsearch-kibana stack.

#### <b> One container to collect them all </b>

So for collecting the logs we went with a fluentd container. we first tried using the fluentd logging driver but then problems arised. First of all we can not start any containers when the fluentd container is not started due to the required `--link`. and secondly, when the fluentd container crashes for some reason, all the logs for the period that is not running are lost.

we have fluentd collecting logs from the default json docker logs, fluentd stores in a position file what logs he already read so when it crashes it just reads the position file and knows what logs it did not send to storage yet.

this was the config file we started with.
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

With this config file all that fluent does is read the json log files from docker. Stores it current position in thos json files in its position file. Then it tags them with a prefix docker. Then when all the logs with a prefix docker (so all of them) are send to the elasticsearch container. This had almoust everything we need except we had no idea what exact container the logs were coming from, the only thing added was the long container id string but that was not what we wanted.
We wanted to have the hostname we originally set when starting the containers 

#### <b> One container to store them all </b>

#### <b> One container to show them all </b>


