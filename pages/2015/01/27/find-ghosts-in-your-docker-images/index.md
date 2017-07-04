---
title: 'Find GHOSTs in your Docker images'
date: 2015-01-27 17:30:17 +0000
tags: 
layout: post
---

A severe security vulnerability in glibc < 2.18, nicknamed [GHOST](http://www.openwall.com/lists/oss-security/2015/01/27/9) was just reported.
Here is a handy one-liner (Debian/Ubuntu only though) to walk through all your Docker images and see if they include a glibc older than 2.18:

```
docker images -q | while read I; do V=`docker run --rm --entrypoint apt-cache $I policy libc6 2>/dev/null | awk ' /Installed/ { print $2"\n"2.18 }'|sort -V|head -1`; if [ -z "$V" ]; then echo "$I not apt based" && continue; fi;  [ "$V" == "2.18" ] || echo "$I is vulnerable"; done
```
