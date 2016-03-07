---
layout: post
title:  "Stage week 1: Getting started with docker"
date:   2016-2-19 14:36:23
categories: Stage
---

<div style="text-align:center"><img src ="../../../../images/docker_logo.png" /></div>

Before beginning our internship at Small Town Heroes, we didn't have any experience with Docker.  
Since Rome wasn't built in a day, we would learn this technology step-by-step, starting with the basics.  



## <strong>1. Setting up Docker on Fedora 23</strong>

The first step in this process was to setup Docker locally on our machine.  
In our case this was our laptop with Fedora 23 as OS.  
Basically we needed two parts, the <strong>docker engine</strong> and the <strong>docker machine</strong>.  

### <strong>Docker engine:</strong>

The docker engine is a client-server application consisting of <strong>three elements:</strong>

1. The docker daemon
2. a REST API
3. and a CLI that talks to the daemon through the REST API

<div style="text-align:left"><img src ="../../../../images/engine.png" /></div>


We installed this very easily by using the <strong>dnf package manager</strong>. For all of these steps, it’s important to be logged in with sudo priviliges.  

