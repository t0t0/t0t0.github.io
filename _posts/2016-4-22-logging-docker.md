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

