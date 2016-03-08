---
layout: post
title:  "Stage week 2: Docker-gen and continuous integration"
date:   2016-2-26 14:36:23
categories: Stage
---
<div style="text-align:center;padding-bottom:25px;"><img src ="../../../../images/stageWeek2/circleci_logo.png" style="max-width:100%" /></div>

Our next step in learning docker was setting up continuous integration. For this we used circleci and some other nifty features. This allowed the other interns whom were develloping applications to push their code and it automaticly depolyed.

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

These makefiles also enable us to nest make commands like `docker_build_run`. This gives us acces to the one command to build and run aswell as both run and build separately.

## <strong>2. circleci </strong>

Here we are not going to discuss how to set up circleci for that we like to point you to the  <a href="https://circleci.com/docs/gettingstarted">documentation on their website</a>.

<div style="text-align:center;padding-bottom:25px;"><img src ="../../../../images/stageWeek2/circleicon.png" style="max-width:100%" /></div>

#### <strong> 1. circleci and docker</strong>

So we had to set up circleci for the 2 projects forwich we had to provide the infrastructure. We ran both these projects on the coreos server we <a href="../../../../stage/2016/02/19/stage-week1-getting-started-with-docker.html">talked about last week </a>. We used an nginx container to proxy to both these projects but more about that later.
<div style="text-align:center;padding-bottom:25px;"><img src ="../../../../images/stageWeek2/circledock.jpg" style="max-width:100%" /></div>

For setting up circleci we defined the tasks in our `circle.yml` file. Because we were using makefiles in our github directory we could use those make commands in the `circle.yml` file. 

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

In our makefiles we don't define any variables. The files with varables are definde by providing the path to the directory in the environment variable `MK_CONFIG` This allows us to use different settings depending on where we want to run the make commands.

The command `sudo cp infra/config/$CIRCLE_BRANCH/* config/` is an implementation of this. In our project directory we use a folder for each branch we are working on, Circleci then when executing this command fills in the `$CIRCLE_BRANCH` with the correct branch it is building for and uses the correct settings for the rest of the build and deployment. 


#### <strong> 2. continuous integration with circleci </strong>

Continuous integration with circleci is setup with the circle.yml file. Each time you push your code circleci automaticly builds and deploys your application in the way you defined in the circle.yml file.

The workflow we went with is shown in the picture below.
<div style="text-align:center;padding-bottom:25px;"><img src ="../../../../images/stageWeek2/contint.png" style="max-width:100%" /></div>

So when one of the developpers pushes to his repo the circleci machine starts.
Firstoff we do a test build on the circleci docker machine. Because currently we there are no unit tests included in the project we only test if the container is still accessible (meaning we didn't get a crash on the deployed code). This is as simple as doing a curl to the container.

When this very basic test succeeds we put our image into a tarbal and push that tarbal to S3 storage. This enables us to deploy this image anywhere we want with relative ease. After this upload to S3 is we have to get this tarbal on EC2. For that we just download the tarbal and replace the current tarbal for that container we have on our EC2 machine.

Then we have to unpack and deploy this image. Because we cant remove and replace an image that is currently in use we first have to remove the old containers running that image. When those old containers are removed we can replace the image and deploy the new containers. In a production environment this would require orchestration but for our dev enviroment this is good enough.




