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