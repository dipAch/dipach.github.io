---
layout: post
title: a date with redis
tags: [redis, deep-dive]
---


Redis needs no introduction. It is a widely deployed data-structure server. In this post today, let's try to build the Redis project from source and play around with it a bit.

## Getting Redis
---

Like most open-source software, redis is also hosted on github. Head on to [https://github.com/redis/redis](https://github.com/redis/redis) and clone the repository, using either the *HTTPS or SSH* options.

## Building Redis
---

In order to build redis and play with it, let's use our docker playground environment that we previously created here - [setup my docker playground]({{site.baseurl}}/dockerize/ "docker up my playground")

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

Once inside the container's shell interface, we can navigate to the mapped redis directory. If you've followed along our previous docker playground blog post, it should be **/webapps/redis/**.

```
bash-5.0# cd /webapps/redis/
bash-5.0# pwd
/webapps/redis
bash-5.0#
```

From inside this redis root directory (inside of your docker container's shell interface), we can execute the **make** command to build redis from the source. Run the below command like so.

```
bash-5.0# make CFLAGS="-ggdb -Og -g3 -fno-omit-frame-pointer"
```

The **CFLAGS** option tells the **make** command to minimize the optimizations done by the compiler (GCC in our case), when building redis. This allows us to transparently inspect the run-time state when using a gdb based debugging session.

## Running Redis
---

Once you've successfully compiled (which will be made obvious by the output of running the above **make** command), we can run the redis-server executable.

The compiled executables (server & client) reside inside the **/webapps/redis/src/** directory as instructed by the Makefile directives.

Run below commands from the redis root, inside of your docker container's shell interface.

To start the server, run the below command.

```
bash-5.0# ./src/redis-server
57:C 05 Sep 2020 14:28:21.294 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
57:C 05 Sep 2020 14:28:21.294 # Redis version=999.999.999, bits=64, commit=c17e597d, modified=1, pid=57, just started
57:C 05 Sep 2020 14:28:21.294 # Warning: no config file specified, using the default config. In order to specify a config file use ./src/redis-server /path/to/redis.conf
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 999.999.999 (c17e597d/1) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 57
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               
                                                                     

       ________        __                                            
      |  ____  |      /  |                                           
      | |    | |     / | |                                           
      | |    | |    /_/| |                                           
      | |    | |       | |                                           
      | |____| |     __| |__                                         
      |________|    |_______|                                        

57:M 05 Sep 2020 14:28:21.300 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
57:M 05 Sep 2020 14:28:21.300 # Server initialized
57:M 05 Sep 2020 14:28:21.300 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
57:M 05 Sep 2020 14:28:21.305 * Loading RDB produced by version 999.999.999
57:M 05 Sep 2020 14:28:21.305 * RDB age 1799556 seconds
57:M 05 Sep 2020 14:28:21.305 * RDB memory usage when created 0.77 Mb
57:M 05 Sep 2020 14:28:21.306 * DB loaded from disk: 0.003 seconds
57:M 05 Sep 2020 14:28:21.306 * Ready to accept connections
```

Don't mind the unfamiliar ascii art, I did it as part of my redis internals learning 😜

To start the client, type below command in another (possibly adjacent) container shell window.

```
$ docker exec -it project-root_go-app_1 bash
bash-5.0#
bash-5.0# ./src/redis-cli
127.0.0.1:6379> set foo bar
OK
127.0.0.1:6379> get foo
"bar"
127.0.0.1:6379>
```

## Conclusion
---

Hurray! You have redis running and ready to roar. Go ahead and try out a few commands. Let's catch up next time and try to get our hands a little bit more dirty.
