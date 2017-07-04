---
title: 'Running a highly available load balancer on Docker'
date: 2014-11-10 19:18:57 +0000
tags: docker infrastructure tech
layout: post
---
For quite some time I felt like I should do a blog - again. Instead of spending time writting rants on facebook or commenting other peoples posting, I should write some blog articles!
Thing is, it's not that easy. Where to start? What should I talk about? What are you interested in reading?

Instead of diving into abstract thoughts about the universe and human kind, I'll just present you how to run a highly available load balancer on Docker!

### Components
For the load balancing part I've choosen haproxy. Since 1.5 it supports SSL termination. Since SSL is expensive, compared to running haproxy without it, we enable multiprocess support by specifying nbproc. Each request now may be handled by a different process. This is a problem for the stats endpoint: Since this endpoint exposes internal state which can't be shared across multiple processes, you only get stats for the process which handles the current request.
To still get metrics for all processes, you need to create a listen endpoint for each process and pin it to that process like this:

```
listen stats01 :8001
  stats uri /
  stats auth admin:foobar23
  bind-process 1
  stats enable

listen stats02 :8002
  stats uri /
  stats auth admin:foobar23
  bind-process 2
  stats enable

listen stats03 :8003
  stats uri /
  stats auth admin:foobar23
  bind-process 3
  stats enable
...
```

With all that in place we can terminate SSL and load balance across multiple hosts but we still need to make this highly available.
There are several ways to do that. I really like ECMP routing but that requires access to the routing layer which I don't have. Then there is IPVS which is a good fit if you need to scale to multiple load balancers but it's also harder and more complex to setup in my opinion.
Because of that, I decided for UCARP which is a implementation of the CARP protocol for linux. CARP is similar to VRRP but patent free and uses cryptography to make it resilient against attackes on the protocol.
When running, it makes sure that there is only one CARP master and executes hooks to add or remove IP adresses.

### Docker Image
I wrapped that all up in a easy to use [Docker image](https://registry.hub.docker.com/u/fish/haproxy/).
To configure haproxy and nginx, you can bind-mount the config location like this:

	$ docker run \
    -v /path/to/haproxy.cfg:/haproxy/haproxy.cfg \
    -v /path/to/nginx.d:/haproxy/nginx.d/ \
    --net=host --privileged fish/haproxy  \
    10.0.1.201 foobar23 [...additional IPs]
*--net=host is required to access the real hosts interfaces. Privileged is necessary to allow the container to bind IPs. Instead of privileged mode, you probably can use --cap-add=NET_ADMIN*

This will run nginx+haproxy+ucarp and make it listen on 10.0.1.201. You can start the same on another host in the same network and ucarp will make sure only one listens on 10.0.1.201. If you kill the active container, the passive one will failover.

Since bind-mounting makes things host dependent, I prefer using the fish/haproxy image as a base image and add my deployment specific configuration in a separated Dockerfile like this:

    FROM fish/haproxy
    ADD  . /haproxy
    RUN  haproxy -c -f /haproxy/haproxy.cfg

*The last RUN serves as a cheap test; It will prevent the build from suceeding if the configs are malformed*

This Dockerfile overwrites haproxy.cfg and nginx.d/ in the image by using the files in the local directory.
Just build it with:

	$ docker build -t my-lb .

And run it as above. This has the downside that you need to rebuild the image on configuration changes and recreate the container to deploy it.

### Service Discovery
If you're using some configuration management system, you can just render the config on the host and bind-mount it. Still better than running you CM inside a container.
A much better solution is to use some kind of service registry for discovery of your backends. I haven't found time for that yet, but I would suggest looking into [consul](http://consul.io), [registrator](https://github.com/progrium/registrator) and [confd](https://github.com/kelseyhightower/confd).
