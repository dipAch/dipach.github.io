---
layout: post
title: docker up my playground
tags: [docker, dev-env]
---

## Intro
---

Docker provides a great way to setup a ready-to-use, throwaway playground environment for software testing and development in general.

For the very same reason let's setup up a multi-purpose docker based environment, where we can build different softwares from the ground up, without polluting our actual host system with dependency installs and the likes.

If you already have a docker setup lying around on your system / workstation, then you can proceed with the other blog posts. Make sure to check if the appropriate system packages are available on your docker image (every project post has a build requirements checklist). If some packages are missing, please checkout the below **Dockerfile** recipe to install the appropriate system packages on to your image or container.

## Do I really need it?
---

I will be using an alpine-linux based docker image, that comes with golang installed. *Why you ask?*

Well, I was learning golang at the time of writing this and wanted a docker image that came with golang installed. You are free to use any linux based docker image and need not even care about having golang installed!

Also, there are few other things that I have added to the below recipe (You may need it, you may not need it). You can cherry-pick or just have as-is. I will be updating the **Dockerfile** recipe, as and when i'll be needing more packages for future projects.

## Where's my Playground?!
---

Before you proceed, make sure you have *docker and docker-compose* available on your system.
If not, then there are plenty of good articles to install docker & docker-compose for your respective Operating Systems. Fire up your favorite search engine, now.

Once you are done with the installation or already have them installed, proceed with the below playground setup.

Below is the tree structure from the **project-root** directory. Name the root directory anything you like:

```
-- project-root
    |-- app
        |-- Dockerfile
        |-- server.go
        |-- Makefile
        |-- docker-compose.yml
```

## What should I put in em'?
---

> **_project-root/app/Dockerfile_**

```
FROM golang:1.14.3-alpine3.11 as dev

RUN apk update && \
    apk upgrade

RUN apk add --no-cache bash make gcc g++ ncurses-dev jpeg-dev zlib-dev libxml2-dev libxslt-dev libffi-dev postgresql-client postgresql-dev openldap-dev geoip-dev build-base readline-dev flex bison libressl-dev openssl-dev libxml2-utils libxslt gdb tcl

RUN apk add --no-cache --virtual .build-deps-testing

RUN apk add --no-cache git openssh mercurial pango-dev swig

RUN apk add --update --repository http://dl-cdn.alpinelinux.org/alpine/edge/main --repository http://dl-cdn.alpinelinux.org/alpine/edge/testing gdal-dev libressl2.7-libcrypto && \
    apk update; apk add linux-headers

RUN adduser -S postgres

RUN mkdir -p /usr/local/pgsql/data && \
	chown postgres /usr/local/pgsql/data

ARG APP_ROOT=/webapps/app
WORKDIR $APP_ROOT
ADD ./ $APP_ROOT/
```

> **_project-root/app/server.go_**

```
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", HelloServer)
    http.ListenAndServe(":8080", nil)
}

func HelloServer(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello, %s!", r.URL.Path[1:])
}
```

> **_project-root/app/Makefile_**

```
run-dev-server:
    go build
    ./app
```

> **_project-root/docker-compose.yml_**

```
version: '3.7'

services:
  go-app:
    build:
      context: ./app
      target: dev
    working_dir: /webapps/app
    command: make run-dev-server
    volumes:
      - ./app:/webapps/app:delegated
      - /Users/dipankar/Workspace/redis:/webapps/redis:delegated
    ports:
      - "8003:8080"
    stdin_open: true
    tty: true
    security_opt:
      - seccomp:unconfined
    cap_add:
      - SYS_PTRACE
      - NET_RAW
      - NET_ADMIN
```

## Build and launch your playground
---

From your project root, where you have the docker-compose.yml file, run below commands:

* **docker-compose build**
* **docker-compose up**

You should be seeing something like below towards the end of a successful build and container startup.

```
Attaching to project-root_go-app_1
go-app_1  | go build
go-app_1  | ./app
```

## Outro
---

Well now you have the base docker environment ready. You can mount any volume from your host system and play with different open-source projects. The sky is the limit.

Let's get started!
