---
title: Deploying a private registry in Docker Swarm
summary: I ran into many headaches while I was trying to deploy a private registry in a Docker Swarm to use with OpenFaaS.  I want to share some of those issues and how I finally got it working.
categories:
- programming
- docker
tags:
- golang
- docker
- docker swarm
keywords:
- golang
- docker
- swarm
comments: false
showPagination: false
date: 2017-10-05T12:00:00Z
draft: true
autoThumbnailImage: true
thumbnailImage: https://docs.google.com/drawings/d/e/2PACX-1vTBlHxSJLzjUG4EA9n8CLdmWfbUDBus2kHOlDt4KuVpVYCGzn6FTMAehfd7VaMx_uujOy3wp-zR_lOl/pub?w=422&h=160
coverImage: https://docs.google.com/drawings/d/e/2PACX-1vTBlHxSJLzjUG4EA9n8CLdmWfbUDBus2kHOlDt4KuVpVYCGzn6FTMAehfd7VaMx_uujOy3wp-zR_lOl/pub?w=1267&h=479
coverSize: partial
#canonicalLink: https://blog.contiamo.com
---

At [Contiamo](https://contiamo.com) I am currently working a project that will eventually integrate with [OpenFaaS](https://www.openfaas.com/).  I am really excited about this project because we will soon bring some serverless magic to data scientists that use Contiamo.  That is, once I figure out how to deploy a private Docker registry inside a Docker Swarm.

<!--more-->

Ultimately, we will be deploying with [Kubernetes](https://kubernetes.io/) in production, but this is not exactly trivial to setup or simple to use as a dev environment. Yes there is [minikube](https://github.com/kubernetes/minikube), but I ultimately had a lot of headaches during my initial experimentation. Docker Swarm, on the other hand, is baked right into Docker for Mac and I find that debugging Docker Services is simplier than K8 Pods. So, it should be an easy target for a simple dev environment.

For this post I have created a sample project called [builderpoc](https://github.com/LucasRoesler/builderpoc).  The basic idea is to have a "build server" that will build and tag an image, push it to a private repo, and then create an OpenFaaS function based on that image.  In this project, for simplicity, it is simply pulling the example `functions/wordcount` image and retagging it as `privatefunc/wordcount`.  There is no actual build step.  If you follow the [README in the repo](https://github.com/LucasRoesler/builderpoc/blob/master/README.md), the final configuration will result in the following diagram

{{< image classes="center" src="https://docs.google.com/drawings/d/e/2PACX-1vTJaWyJnc6i6dzY9mFR6stePKM7HlgIfEP_TrQmLyvFDROufwuIcpkbBAU-d4zs4S8DhTbcY3jzhWQP/pub?w=960&h=720" thumbnail="https://docs.google.com/drawings/d/e/2PACX-1vTJaWyJnc6i6dzY9mFR6stePKM7HlgIfEP_TrQmLyvFDROufwuIcpkbBAU-d4zs4S8DhTbcY3jzhWQP/pub?w=480&h=360" title="The basic swarm structure">}}

The natural thing to do is to start with a single machine swarm.  If you have already played with OpenFaaS, then you already have one and it is probably called `moby`.  The next natural thing is to google for `docker swarm private registry` or something similar.   If you do this you will probably be very excited to see that there are [official docs](https://docs.docker.com/registry/deploying/) that will walk you through the process of running a Docker registry.  It will even show you how to run it as a swarm service, jackpot!  This all works _roughly_ as expected. Following the tutorial, my initial docker-compose file for this project ended up looking like this:

```md
version: "3.2"
services:
    gateway:
        volumes:
            - "/var/run/docker.sock:/var/run/docker.sock"
        ports:
            - 8080:8080
        image: functions/gateway:0.6.3
        networks:
            - functions
        environment:
            dnsrr: "true"
        deploy:
            placement:
                constraints:
                    - 'node.role == manager'
                    - 'node.platform.os == linux'

    privateregistry:
        image: registry:2
        volumes:
            - registry:/var/lib/registry
        ports:
            - 5001:5001
        networks:
            - functions
        environment:
            REGISTRY_HTTP_ADDR: "0.0.0.0:5001"
            REGISTRY_HTTP_TLS_CERTIFICATE: /run/secrets/builder_domain.crt
            REGISTRY_HTTP_TLS_KEY: /run/secrets/builder_domain.key
        secrets:
            - builder_domain.crt
            - builder_domain.key
        deploy:
            placement:
                constraints:
                    - 'node.role == manager'
                    - 'node.platform.os == linux'

    server:
        image: theaxer/builderpoc:latest
        ports:
            - 9090:9090
        networks:
            - functions
        command: [
            "builder",
            "-port=9090",
            "-host=0.0.0.0",
            "-registry=privateregistry:5001"
        ]
        volumes:
            - "/var/run/docker.sock:/var/run/docker.sock"
        deploy:
            placement:
                constraints:
                    - 'node.role == worker'
                    - 'node.platform.os == linux'
secrets:
    builder_domain.crt:
        file: ./certs/domain.crt
    builder_domain.key:
        file: ./certs/domain.key
volumes:
    registry:

networks:
    functions:
```


_Quick aside_, it took me awhile to figure out how to best test that images were actually being pushed into the registry.  The trick is to use the registry API directly via `curl`.  If you follow the [Docker tutorial](https://docs.docker.com/registry/deploying/) and have a registry bound to `localhost:5001`, then you can use

```
$ curl http://localhost:5001/v2/_catalog
{"repositories":[]}
```
to see a list of repositiories in your registry.  It should be empty to start and then grow with each new image you push.  For my proof-of-concept project, I bound everything to `builderpoc` and because it is not on localhost, I used self-signed certs, so the above command becomes

```
$ curl -k https://builderpoc:5001/v2/_catalog
{"repositories":[]}
```
Using the API directly is much simpler to consistently test versus changing your Docker env variables and trying to test this via the Docker cli.


Back to the project, if you clone the project run the following commands

```sh
$ git clone git@github.com:LucasRoesler/builderpoc.git
$ cd builderpoc
$ go get github.com/LK4D4/vndr
$ vndr
$ docker swarm init
$ docker stack deploy builderpoc --compose-file docker-compose.yaml
```

You will have something like this

{{< image classes="center" src="https://docs.google.com/drawings/d/e/2PACX-1vRD8jdAm2L9CPGmPnslEop6JNzxrwAzBAAKYE-ySo5kWjvesgwaI34hGEILX2kc7tzfz1WOM7SJYU1j/pub?w=960&h=720" thumbnail="https://docs.google.com/drawings/d/e/2PACX-1vRD8jdAm2L9CPGmPnslEop6JNzxrwAzBAAKYE-ySo5kWjvesgwaI34hGEILX2kc7tzfz1WOM7SJYU1j/pub?w=480&h=360" title="The failed single node swarm structure">}}

and it will appear to work as expected. It will pull and tag an image, it will even deploy a function, but ... it isn't actually pulling the image during the deploy.  In this single node configuration the `server` and the `gateway` service are both mounting the same `docker.sock` and therefore share same Docker image cache. The whole swarm is sharing the same cache, so when the OpenFaaS `gateway` starts the swarm service for this new function, it is just referencing the cached image.  In fact, if you comment out line 101 of the `builder/main.go` and stop the registry service, you should see everything "work".

When I noticed this in my own project, I decided that I needed to develop in a multi-node environment. So, I issue a few `docker-machine create` and various other commands, all documented and scripted in the [swarm script](https://github.com/LucasRoesler/builderpoc/blob/master/swarm), and I have my swarm. For the `buildpoc` all you need to do is

```sh
# setup a two node swarm
./swarm init
# set the docker env so that our cli points to the manager1 docker engine
eval $(docker-machine env manager1)
# build the server image
./swarm build
docker pull registry:2
# copy server image to the worker node
docker save theaxer/builderpoc:latest | docker-machine ssh worker1 "docker load"
# now it can be deployed
./swarm deploy
```
and you should have a swarm that looks like this:

{{< image classes="center" src="https://docs.google.com/drawings/d/e/2PACX-1vTJaWyJnc6i6dzY9mFR6stePKM7HlgIfEP_TrQmLyvFDROufwuIcpkbBAU-d4zs4S8DhTbcY3jzhWQP/pub?w=960&h=720" thumbnail="https://docs.google.com/drawings/d/e/2PACX-1vTJaWyJnc6i6dzY9mFR6stePKM7HlgIfEP_TrQmLyvFDROufwuIcpkbBAU-d4zs4S8DhTbcY3jzhWQP/pub?w=480&h=360" title="The basic swarm structure">}}

Using the current `builderpoc` scripts, all of this will work.  However, with my initial compose file that I showed above, it blows up in my face.

{{< image classes="center nocaption" src="https://i.makeagif.com/media/10-20-2014/jou739.gif" >}}

In particular, the push from the `server` service to the `privateregistry` was giving me errors like

```
Get https://privateregistry:5001/v2/: dial tcp: lookup privateregistry on 10.0.2.3:53: no such host
```

You can reproduce this in the `buildpoc` with a simple change at [line 50 of the compose file](https://github.com/LucasRoesler/builderpoc/blob/master/docker-compose.yaml#L50) from
```md
    "-registry=localhost:5001"
```
back to

```md
    "-registry=privateregistry:5001"
```

The core of the issue is that we have mounted the `docker.sock` from our host vm into the container for the `server`.  This means that all of the docker commands, e.g. `push`, `pull`, and `tag`, are actually running on the host vm, including DNS resolution.  But service discovery for a stack is only available to the services inside the stack, not to the host vm.

Switching back to `localhost:5001` works because [26 lines above that one](https://github.com/LucasRoesler/builderpoc/blob/master/docker-compose.yaml#L24), we have bound the registry ports to `5001`. So, each host machine now resolves `localhost:5001` to the internal port `5001` of `privateregistry`, much like in a standard docker-compose configuration. Of course, this all breaks if we then remove that port binding.

This took me several days to debug and then finally fix. I had considered many complicated things, from some nasty bash scripting to update host files to adding in Consul.  I have to give a lot of thanks to pav at [Dots and Brackets](http://codeblog.dotsandbrackets.com/) for a small little update in his own blog post about [Using private registry in Docker Swarm](http://codeblog.dotsandbrackets.com/private-registry-swarm/).  He has a short little update in the middle of the post:

> Update: as smart people mentioned, if registry is going to be pushed to and pulled from only from within the cluster, we still can go with HTTP – localhost is shared within the cluster

I had to read it several times because it seemed too simple.  Ultimately it was the correct fix and I thank pav for sharing that little tidbit.
