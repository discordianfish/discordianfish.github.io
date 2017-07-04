---
title: 'Using docker run --net=container:XX ... to debug network issues'
date: 2014-11-13 23:04:22 +0000
tags: 
layout: post
---
Sharing the network namespace with a existing container is a less known feature of Docker. If you run:

	docker run --net=container:my-existing-container ...

Your container runs in the same network namespace, sharing the same network configuration as `my-existing-container`.

This is very useful for debugging purposes: I just wanted to verify the current outgoing connections from a container in our infrastructure and realized that due to the network scoping that's not easily possible. You can use `ip netns exec` but that assumes you have the namespace mounted to `/var/run/netns`. You can symlink things around but it's ugly. Much nicer:

	docker run -t -i --net=container:my-existing-container --rm ubuntu netstat -anp
