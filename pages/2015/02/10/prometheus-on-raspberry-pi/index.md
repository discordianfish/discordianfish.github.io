---
title: 'Prometheus on Raspberry Pi'
date: 2015-02-10 13:18:12 +0000
tags: 
layout: post
---
# Prometheus
[Prometheus](http://prometheus.io) is a new open-source service monitoring system and time series database written in Go.

Check out the [announcement](https://developers.soundcloud.com/blog/prometheus-monitoring-at-soundcloud) and my article about [monitoring Docker Containers with Prometheus](/2015/01/26/monitor-docker-containers-with-prometheus) if you don't know what I'm talking about.

# My Stack
![RaspberryPi](http://upload.wikimedia.org/wikipedia/commons/thumb/d/d2/Raspberry_Pi_Photo.jpg/800px-Raspberry_Pi_Photo.jpg)
I stopped running my own full blown server(s) a while ago. Nowadays I just have a old Raspberry Pi at home and two tiny DigitalOcean instances to host this blog and for general R&D stuff.

But I still want to monitor all this and, as showed earlier, Prometheus is the way to go. So I could now spin up another DigitalOcean instance and pay another $5/Mo, but given how cheap I am, I'd rather want to run it on my Raspberry Pi.

# Cross-Compiling Go
First you need to build Go with support for your target OS and architecture. If you installed Go from sources, you can do this by running:

```
cd $GOROOT/src
GOARCH=arm ./make.bash
```

With pure Go, cross compilation is trivial. This example:

```
package main

import "fmt"

func main() {
	fmt.Println("Hello World")
}
```

can be cross-compiling for arm with `GOARCH=arm go build test.go`.

Now things get much more complicated once you use CGO, meaning Go code calling C functions.

The Prometheus server uses the [Prometheus Go Client Library](https://github.com/prometheus/client_golang) to provide metrics about itself. This client library uses [prometheus/procfs](https://github.com/prometheus/procfs) which requires CGO to get process metrics from procfs. Cross-compiling this for Raspberry Pi is a pain. ARM != ARM, there are several variants and when I tried to cross-compiling Prometheus with CGO, it just lead to segfaults or invalid instructions. It should be possible to cross-compile it with the CGO dependency, it's just painful and I quickly gave up.

Fortunately, we found an easy way to remove the dependency on procfs if CGO is disabled: https://github.com/prometheus/client_golang/commit/93d11c8e35ffcd969fd881efe1873e715a6ef93b. This removes some useful process level metrics about prometheus itself, but it makes cross compiling prometheus is as easy as in the example above:

	cd $GOPATH/src/github.com/prometheus/prometheus
    go get -u # Update your dependencies
    GOARCH=arm go build -o prometheus.arm

Since cross-compilation by default disables CGO, this builds a statically linked prometheus binary ready to be run on a Raspberry Pi. If you're running a newer Pi, you can set GOARM to the version of your ARM processor. See [this](https://code.google.com/p/go-wiki/wiki/GoArm#Supported_architectures) for supported architectures.

# Performance
![goroutines](/content/images/2015/Feb/rpi_gorouting.png)
I didn't have time to benchmark it yet, but it seems to perform much better than I expected. Right now, it only scrapes itself, giving us ~250 time series. After running it for a few hours it already collected 137806 samples. Graphing a simple time series like `process_goroutines` for the past hour takes between 150-170ms. The probably most expensive operation you can do (and definitely shouldn't do on a production system) is graphing *all* time series by executing `{job=~".*"}` this returns in about 30s on the Raspberry Pi.
![goroutines](/content/images/2015/Feb/rpi_all-1.png)

# Downloads
**Update:** There are now official images available:
https://prometheus.io/download/

### Checksums
```
$ shasum prometheus.armv?.gz
db42a3f568bbab1d8a7d183b336d0b50dccc80b6  prometheus.armv5.gz
b6fdd3a77e16359631a0daaf9c06cd93a9948932  prometheus.armv6.gz
8d3e6580ee4ed3d10b93d60c13009156a5600193  prometheus.armv7.gz

$  sha256sum prometheus.armv?.gz
e206202bb07cc139eaabac4c51c07f0a332337eb331bd79f19294c558fb3de62  prometheus.armv5.gz
99d864e4fee8ded6b0f9c117f839fdf0fa9d12fadfe27b98090bee04b88c48c0  prometheus.armv6.gz
baf550448174198c57f3a28e2b86d49ba523e5b15d9d4c087267021e4785299b  prometheus.armv7.gz
```
