---
title: Don't manage configuration unless you have to.
date: 2015-08-31 13:45:56 +0000
tags: immutable-infrastructure configuration-management
layout: post
---
<a data-flickr-embed="true" data-header="false" data-footer="false" data-context="false"  href="https://www.flickr.com/photos/tuinkabouter/1884416825/in/photolist-3Sw8tg-4pYNoQ-5kuQnX-5gVeH1-sVYLE3-a3DqaD-9gzdMP-9Uspfa-2Xkkg8-8oZ8r-5aUCED-e4npy8-bn7jhK-ipQLm-4pZj1h-4xV1jk-5kuMV6-9sUBkF-b8nM3X-7tsJDh-9hY5PC-dUvDCG-4QZswm-4io4Y8-812A6f-qprAf-5ojVcV-5wUanj-9kNyLj-5GXFGx-5YFqsj-6VJnS6-dUmEaq-99qjSa-4QZsvG-4QZsuW-4QZsu7-4QZsuo-eL5jpe-4EWyKW-CXs86-etDrFm-4M5joL-jpb3gs-5knnT4-GAgoa-6d2TiA-63t6NF-7TRs3F-51XCQG" title="Jungle"><img src="https://farm3.staticflickr.com/2037/1884416825_521e525758_o.jpg" width="1024" height="576" alt="Jungle"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

At my first job, there was no code driven infrastructure at all. A nightmare from today's perspective.

The next job, there were some perl scripts to massage systems in a imperative way.

Then, at SoundCloud, I used modern configuration management for the first time. SoundCloud heavily invested into Chef which was, at some point, used to drive almost every aspect of the infrastructure. A new service? Add a cookbook.

Compared to a world without any proper infrastructure automation this obviously was a great step in the right direction but it was a growing pile of technical debt and constant source of bikeshedding around how to actually use it.

Maybe you don't have those problems. Maybe you have strict guidelines on how to use your configuration management or some chief architect to tame the chaos but I argue that lot of larger companies with agile and independent teams run into similar problems. If you disagree, please leave a comment.

### Issues with Chef
There are several categories of problems with this setup. Some are organizational nature, like the way it was ab(used) to drive the complete infrastructure. There are also very specific issues with chef, like its internal complexity both from a operational perspective as well as from its complex interface. Things like 15 unintuitive precedence levels for node attributes to its multistep execution flow and leaky cookbook abstractions it's often frustrating to use and accumulates a lot of technical debt due to it being hard to test and refactor without potentially breaking your infrastructure.

### Alternatives - or the lack thereof
That being said, there isn't an obvious better alternative. In the meanwhile I used ansible and salt which have their very own problems. Even though they try to be less complex than chef, they heavily depend on template driven metaprogramming and struggle with proper code reuse and testability similar to chef.

Over the years I came to the conclusion that configuration management in it's current form as used in reality has some fundamental design issues.

### Design challenges
![vortex](/content/images/2015/Aug/Airplane_vortex_edit.jpg)
The idea of defining a (distributed) systems state and mutating it to eventually converge to the desired state is sound but the interface the distributed systems provide is simply too complex.

All the mentioned configuration management systems use some agent or ssh access to execute commands on the systems, similar to the imperative design of user interfaces: You run commands in a specific order to modify the state of the systems.
But since there is no unified interface to configure applications, how to achieve a specific state is highly dependent on the application.
Configuration management systems try to solve this by abstracting a "thing" in a system and give it a clear interface with some idempotent functions to move it into a given state like `installed`. Whether it's called chef cookbook, salt state or (the most misleading name) ansible role.
If the "thing" is your web application, this might work very well but in reality you often have to configure third party applications that have subtle dependencies on specifics of other "things". Or you have low level system configuration which affects and depends on other things installed. At this point, the abstractions usually break and you often end up introducing site specific changes to what is suppose to be reusable, generic components.

The lack of a generic system configuration interface that can be used to configure every aspect of a system, imposes a lot of complexity on the configuration management.

### Split configuration based on life cycle
<a data-flickr-embed="true" data-header="false" data-footer="false" data-context="false"  href="https://www.flickr.com/photos/chloeophelia/6633874815/in/photolist-b7dkRZ-j6g2XM-9hrBqG-aTDRwD-99nLPX-tm1pU-bn3u8p-qHs14o-qCjhgq-4HCs5T-r9mjBJ-9MyabW-q4629m-oCKk2V-7zyviy-7HivZw-8Wzh3Y-k2UGmz-iuChmC-dKHA-qe133d-itoizS-83TM-oHsMP1-bstTCp-iY9u4G-jdRX2T-5SY4oB-7wmEBq-Cv7Z2-t2Hab-7ofMcb-99Eqbu-r5sCGR-93jB3F-qyomNx-riyS2i-dNMQsp-91DhgW-4ahpVw-qHSWG3-keBc61-5SJkro-qixdjy-dSyxEq-8nmzni-4aKkPV-f2gfQ-q96Qs1-iyGqwU" title="frozen in time"><img src="https://farm8.staticflickr.com/7142/6633874815_5d4ecf8b71_b.jpg" width="1024" height="684" alt="frozen in time"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

As long as there is configuration, there is configuration management. You will always have some form of desired state you want your infrastructure to be in. The question is not if but how to manage configuration. Since with configuration that changes rarely less things can go wrong, I believe the best way is so identify different configuration life cycles, find the right solution to manage this kind of configuration while compromising dynamically for correctness.


#### Build/install time configuration

If the lifetime of a single host or container image is lower than the lifetime of a configuration option, it often makes sense to move this configuration to install/build time. Since no change can happen during the life cycle of the host or container it's easier to reason about the infrastructure since change to this set of configuration can be ruled out as reason for a given observation.

Moving configuration to install time might mean making your bare metal installer preseed some configuration or building a static OS image for your cloud provider. The point is to bake in this configuration and just rebuild when changes are necessary.

##### Examples
- Operating system release
- Partition schema
- Hostname
- Installed packages / core services
- OS level configuration (sysctl, ssl keys)

All configuration that is same across all environments (dev/test/prod) should be considered to be hardcoded where environment specific configuration (credentials, URLs / service identifiers) should be passed in on runtime so you can build, test and deploy the same static artifact.

What in reality gets hardcoded is a case by case decision and depends on a lot of factors. On a bare metal infrastructure where all services are deployed straight to the host, there will be a lot of configuration that is site-wide and environment agnostic but simply is changed that often that reinstallation of the whole host isn't feasible.

But in a containerized infrastructure you have host image and container image life cycles. There is little host configuration, so usually it all can be hardcoded. Even if reinstalling the host takes 20 minutes, if it only happens every few weeks and is fully automated, it's probably fine.
Building the container images in a continuous deployment pipeline might just take a minute from a change until the changes are deployed, so here again it's feasible to bake in all suitable configuration.

#### Start time configuration
<a data-flickr-embed="true" data-header="false" data-footer="false" data-context="false"  href="https://www.flickr.com/photos/storm-crypt/326228715/in/photolist-uQ1r2-t62c-jjzxu-pzhfMZ-7pBiCr-gKqR8Z-cqHKJS-hZsSz-522oaS-b3uw2-bXcpyw-5zdLNo-6wvjpw-nv1KL3-3aNjwL-cpBpTC-dbcLKt-4o3Qkk-9pyoHg-6dvJoB-hzvCsP-4o7SHY-aCL42j-4tGWBK-4wxATP-bX13ov-8MgTTH-cZKwvf-iKDCAd-jRyyK1-nb69S1-kvg2qy-kvdS8F-kveev8-kvdGta-kvdFzg-kvdxKi-kve2eV-bpcAmi-rbVVC-4gi4N-4rA5gu-aazkxv-o2XySp-dxnLQi-2oa9re-eC2cX3-8JA67v-8KYUaA-8JA6aB" title="Sorting Facility"><img src="https://farm1.staticflickr.com/135/326228715_dea4917fda_b.jpg" width="1024" height="768" alt="Sorting Facility"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

Especially environment specific configuration like credentials should be passed to the services by whatever deploys your application.
Even though a lot people still deploy their applications with configuration management instead of cluster schedulers, I'm convinced that will change in the next few years. Whether you're using [Mesos/Marathon](https://mesosphere.github.io/marathon/), [kubernetes](http://kubernetes.io/) or [Omega](http://eurosys2013.tudos.org/wp-content/uploads/2013/paper/Schwarzkopf.pdf) the high level concepts are similar: You define your application and the scheduler decides based on the available resources where to run it.
Whether services are deployed by config management systems or the cluster scheduler, since it's starting services, it's the right place to pass configuration to your service. Instead of writing configuration files, [12factor style configuration](http://12factor.net/config) is usually better suited.

#### Runtime configuration
Instead of configuring your systems on a regular interval with some configuration management daemon, it's often a better pattern to have the application or a wrapper around it determine the configuration on runtime.
Instead of making configuration management orchestrate various services or instances, it's more robust and arguably less cognitively challenging to consider the service as it's own independent entity.
This only works well if site-wide configuration is built in. Determining all this on runtime leads to similar complexity as we have with full blown configuration management today.

### Conclusion
<a data-flickr-embed="true" data-header="false" data-footer="false" data-context="false"  href="https://www.flickr.com/photos/ph0t0s/96911576/in/photolist-9yGrf-rf1xpg-iV3Hti-9yGuq-9mJACS-dF5fsX-4CYijn-96MGD7-4CYinT-rcHmFs-e3mfmk-7paoHV-e3rYs3-ebixTV-e3miwv-e3mehH-ebz9br-ebz6nv-9yGn2-maubuc-4eMsoq-gascSh-j448Si-bUvoKX-7tWwN9-8oTPht-fw2NWm-34Ydj3-zMTMt-9NFVNX-4q1pdk-9eX7xW-e72dfP-eb7MBt-4xMiAY-bznrxn-ze6qt-5GtxU5-bhCNRp-fw2NjN-fvMuU4-ebEQHU-fvMxdc-e3rZzd-ebiz6V-fw2Qn7-9NEtYQ-9NGKrb-u79vMN-9NGokq" title="partly random"><img src="https://farm1.staticflickr.com/36/96911576_1a57864a0b_b.jpg" width="1024" height="768" alt="partly random"></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>
Isolating change to specific points in the life cycle of systems and services reduces the complexity of runtime configuration and simplifies the mental model when reasoning about the infrastructure.
