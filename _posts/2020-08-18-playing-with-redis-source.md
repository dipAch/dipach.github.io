---
layout: post
title: A date with Redis
tags: [redis, deep-dive]
---


Redis needs no introduction. It is a widely deployed data-structure server. In this post today, let's try to build the Redis project from source and play around with it a bit.

## Getting Redis
---

Like most open-source software, redis is also hosted on github. Head on to [https://github.com/redis/redis](https://github.com/redis/redis) and clone the repository, using either the *HTTPS or SSH* options.

## Building Redis
---

In order to build redis and play with it, let's use our docker playground environment that we previously created here -> [setup my docker playground]({{site.baseurl}}/dockerize/ "docker up my playground")

Most of the dependencies required to build and run the project, are already setup as part of the docker based playground.

Map the volume where redis was cloned in your host system, onto your **docker-compose.yml** file as below:

```
version: '3.7'

services:
  go-app:
    ...
    ...
    ...
    volumes:
      ...
      ...
      - /Users/dipankar/Workspace/redis:/webapps/redis:delegated
    ...
    ...
    ...
```

Once the mapped volume is available in your docker instance, we can build redis from the ground up. You will most linkely need to re-start your container after changing the above **docker-compose.yml** file.

With all of the above done, we can now connect to the shell interface of our running image instance (a.k.a the container). Type below command in your terminal window.

```
$ docker exec -it project-root_go-app_1 bash
```

Once inside the container's shell interface, we can navigate to the mapped redis directory. Now if you've followed along our previous docker playground setup blog post, it should be **/webapps/redis/**.

From inside this redis root directory, we can execute the **make** command to build redis from the source. Run the below command like so,

```
bash-5.0# make CFLAGS="-ggdb -Og -g3 -fno-omit-frame-pointer"
```

The **CFLAGS** option tells the **make** command to minimize the optimizations done by the compiler (GCC in our case), when building redis. This allows us to transparently inspect the run-time state when using a gdb based debugging session.
