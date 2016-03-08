---
layout: post
title:  "Internship week 2: Docker-gen and continuous integration"
date:   2016-2-26 14:36:23
categories: Internship
---
<div style="text-align:center;padding-bottom:25px;"><img src ="/images/stageWeek2/circleci_logo.png" style="max-width:100%" /></div>

Our next step in learning docker was setting up continuous integration. For this we used CircleCI and some other nifty features. This allowed the other interns whom were developing applications to push their code and it automaticly deployed.

## <strong> 1. Makefiles </strong>

The first thing we used were makefiles. Here we could define our own make commands so we needn't type out the full commands. An example of one of these makefiles can be found below.

```makefile
#####
# vars
####

IMAGE_NAME=demo
IMAGE_VERSION=latest
PORTHOST=80
PORTGUEST=80
DOCKERFILEPATH=./Dockerfile

####
# concatenate vars
####

# full file name for tarbal
TARNAME = $(IMAGE_NAME).tar
# full image name
IMAGE = $(IMAGE_NAME):$(IMAGE_VERSION)

####
# Make commands
####

docker_build:
	docker build -t $(IMAGE) -f $(DOCKERFILEPATH) .

docker_run:
	docker run -d -p  $(PORTHOST):$(PORTGUEST) $(IMAGE); sleep 10

docker_build_run: docker_build docker_run

docker_save:
	docker save -o $(TARNAME) $(IMAGE)
```
<br />

These makefiles also enable us to nest make commands like `docker_build_run`. This gives us access to the one command to build and run aswell as both run and build separately.

## <strong>2. CircleCI </strong>

In this section we are not going to discuss how to set up CircleCI. For that we like to point you to the  <a href="https://circleci.com/docs/gettingstarted">documentation on their website</a>.


#### <strong> 1. CircleCI and docker</strong>

So we had to set up CircleCI for the 2 projects for wich we had to provide the infrastructure. We ran both these projects on the CoreOS server we <a href="internship/docker/2016/02/19/stage-week1-getting-started-with-docker.html">talked about last week </a>. We used an nginx container to proxy to both these projects but more about that later.
<div style="text-align:center;padding-bottom:25px;"><img src ="/images/stageWeek2/circledock.jpg" style="max-width:100%" /></div>

For setting up CircleCI we defined the tasks in our `circle.yml` file. Because we were using makefiles in our github directory we could use those make commands in the `circle.yml` file. 

```YAML
machine:
  services:
    - docker

dependencies:
  pre:
    - sudo apt-get update; sudo apt-get install s3cmd 
  override:
    - sudo mkdir config
    - sudo cp infra/config/$CIRCLE_BRANCH/* config/
    - MK_CONFIG=config make docker_build

test:
  override:
    - MK_CONFIG=config  make docker_run_node
    - MK_CONFIG=config  make docker_curltest

deployment:
  dev:
    branch: master
    commands:
      - MK_CONFIG=config make docker_save
      - MK_CONFIG=config make s3_config
      - MK_CONFIG=config make s3_upload
      - MK_CONFIG=config make docker_deploy
```
<br />
As you can see the `circle.yml` file becomes a lot more managable and readable using the make commands.

In our makefiles we don't define any variables. The files with variables are defined by providing the path to the directory in the environment variable `MK_CONFIG`. This allows us to use different settings depending on where we want to run the make commands.

The command `sudo cp infra/config/$CIRCLE_BRANCH/* config/` is an implementation of this. In our project directory we use a folder for each branch we are working on, CircleCI then when executing this command fills in the `$CIRCLE_BRANCH` with the correct branch it is building for and uses the correct settings for the rest of the build and deployment. 


#### <strong> 2. continuous integration with CircleCI </strong>

Continuous integration with CircleCI is set up with the `circle.yml` file. Each time you push your code CircleCI automatically builds and deploys your application in the way you defined in the `circle.yml` file.

The workflow we went with is shown in the picture below.
<div style="text-align:center;padding-bottom:25px;"><img src ="/images/stageWeek2/contint.png" style="max-width:100%" /></div>

So when one of the developers pushes to his repo the CircleCI machine starts.
First off we do a test build on the CircleCI docker machine. Because currently there are no unit tests included in the project, we only test if the container is still accessible (meaning we didn't get a crash on the deployed code). This is as simple as curling to the container's ip address.

When this very basic test succeeds we put our image into a tarbal and push that tarbal to S3 storage. This enables us to deploy this image anywhere we want with relative ease. After the upload to S3, we have to get this tarbal on our EC2 machine. Then we just download the tarbal and replace the current tarbal we have on our EC2 machine for that container.

Next we have to unpack and deploy this image. Because we can't remove and replace an image that is currently in use we first have to remove the old containers using that image. When those old containers are removed, we can replace the image and deploy the new containers. In a production environment this would require orchestration but for our dev enviroment this is sufficient.


## <strong>2. Docker-gen for nginx </strong>

When we started using CircleCI for auto deployment a problem arised. Every time code got pushed and deployed to our CoreOS server, new containers with new ip-adresses were created. Because of this we had to go into the nginx container to adjust the configuration. This wasn't very continuous so we had to find a solution for this.

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
