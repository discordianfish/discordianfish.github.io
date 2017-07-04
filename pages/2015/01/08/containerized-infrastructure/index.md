---
title: 'Containerize your Infrastructure'
date: 2015-01-08 14:43:00 +0000
tags: docker containers abstraction immutable-infrastructure
layout: post
---
### The Hype
Everyone is talking about containers.
Containers are a great way to bundle your application with all dependencies and isolate it from other applications running on the same host.
That's all great but we never cared that much about containerization even though it exists for quite some time: chroot() was born 1979 and IBM had tech similar to containers on S/370 over 40 years ago.

![Grumpy Ops](/content/images/2015/Jan/mainframe.jpg)

So why all the hype right now? It's not like we just started with all that.
There are various use cases for containers but here I'll focus on use them as building blocks in modern IT infrastructures.

### Infrastructure evolution
If we look back 10 or maybe 15 years, things looked very differently.
It's not particular news that software eats the world and every company today is, at least partially, an IT company. But back in the 90s, IT was still something for rather specialized companies. Finance, Insurance, Government, Health care. Basically companies juggling lots of numbers.

Big and monolithic software and services designed by a waterfall model. Release cycles in the terms of months and large Ops departments managing them. All scaled mostly vertically by just buying bigger boxes.
Things happened really slow back in the days. Despite the fact that there are still lot of companies stuck back then, even in traditionally more conservative businesses like finance you see [a radical shift towards agile development methodologies](https://www.youtube.com/watch?v=6FPXbQ2WpAM). Why this is a crucial move and I don't belive any company will survive without being "agile" is probably a topic for another blog article.

Fact is, today things are different. To be agile means to be fast, to be able to adapt to a changing environment quickly. The result is a infrastructure optimized for change. Instead of having monolithic application where every change needs to be coordinated with every stakeholder, such infrastructures consist of dozens of loosely coupled (micro)services with independent teams iterating on them, scaled horizontally across hybrid infrastructures with some workload on bare metal, other on cloud instances.
Instead of a release every few weeks, it's not uncommon to deploy multiple times a day fully automated by CI/CD pipelines.

### Challenges
Now we have more services, more changes, more systems and arguably less time to market. Fact is, it's incredibly hard to really understand such infrastructure. Just think about what it means to manage a single service. To deploy it, monitor it, update it.  Now multiply by a few dozens. The millions of knobs and billions of permutations. Nobody can manage that.

![Automate all the things](/content/images/2015/Jan/automate.png)

So what do we do? We automate.
Historically, you had some (shell) scripts to manage things in a imperative way. Whether it's creating some users or deploying an app.

Nowadays almost everyone uses some sort of configuration management systems. The mother of modern configuration management is probably [CFEngine](http://en.wikipedia.org/wiki/CFEngine), even though the idea goes back to the [50s](http://en.wikipedia.org/wiki/Configuration_management#History).
As oppose to the imperative approach, the idea here is declarative. You describe the desired state of your infrastructure and have some logic in place to actual tune all those knobs. Running you config management tool *converges* your infrastructure towards the desired state. In theory, no matter what state your system is in right now, the configuration management system will converge the system to the desired state eventually.

A big advantage over the imperative approach. But in reality, there are just too many knobs and unexpected side effects for this to be very reliable.
In the end, state convergence helps with managing change in your infrastructure but it doesn't make the infrastructure itself simpler. You still first need to understand all the dependencies and interactions between your knobs. You need to somehow reduce the complexity, the number of knobs.

### Managing Complexity
The most important aspect of managing any kind of system is to understand it. To understand how various components influence each other, which parts are healthy and which not and ultimately understand the effects of every possible change to your infrastructure.
Containers help to manage complexity.

#### Abstraction
Every time the cognitive overhead imposed by a system growing in complexity reaches a certain threshold, people start to *abstract* things.
It probably started in human communication. We don't have to be biologists to talk about cats. We don't need to understand every intrinsic *implementation detail*. Same for classes or modules in programming languages. And the same is true for containers: They are an abstraction around a specific application, abstracting all its implementation details and provide a generic interface shared by all other applications. All the knobs are still there, but you hide those you don't care about.

Given the common interface among all your applications, all your systems are doing is running containers. Your DB servers? Running containers. Your application servers? Running containers. Which means instead of having a bunch of servers for different use cases, you only have servers running containers. This allows for systems like cluster schedulers to abstract away the servers and just present a block of resources to run containers on.

You might argue that the complexity isn't gone, it's just hidden and that's true. But the same is true for abstractions in linguistics. Stereotypes are abstractions as well and they often do not accurately reflect reality. Just take Gender as an example. It's important to keep in mind that [all abstractions are leaky](http://www.joelonsoftware.com/articles/LeakyAbstractions.html) and you still need to understand the implementation details or layers below. But you don't need to think about them all the time.

#### Ownership
Especially if a company grows fast, bottlenecks arise. If your company started with five friends sitting around a desk and bouncing ideas back and forth, everything is fine. Every problem is apparent and can be solved quickly by whoever has the necessary time and skills.
Once you grow larger, this isn't possible anymore. Suddenly you need to send mails to request resources from other teams or even pull out the *global write lock* sledgehammer called "meeting" to coordinate change.
Containers can help by enforcing a clean separation of concern between the teams providing resources by operating systems to run containers and the teams running containers on them. If your develpment team needs to deploy a new application, it's just another container. They don't care where it runs. If the infrastructure team needs to provide more resources, they just provide a new system to run containers on.

#### Immutability 
The great thing about computers is their versatility. It's a general purpose device which can be programmed to solve specific tasks. You can install various software on your server, remove it again, change configuration files, set OS settings. You can change the *state* of your machine whenever you want. Remember, this is how configuration management handles change in your infrastructure. Mutating the current state until you get to your desired state. The complexity arises from the fact that every state mutation leads to a new initial condition for all further operations in your system. Configuration management tries to account for all this, but in reality you often end up in states where your assumptions don't apply anymore and you need to manually intervene.
Containers are usually, and as implemented by Docker, based on immutable images. Build- and runtime are clearly separated. Once built, your application, all its dependencies and even parts of your configuration are baked into your images. Even though [Docker doesn't enforce it yet](https://github.com/docker/docker/issues/7923), conceptually by default the container doesn't change. From deployment to removal, it stays the same and there won't be a new initial condition which need to be considered for further operations on the system.
Of course, your infrastructure isn't immutable. You most likely store and process data and with every deploy the state of your infrastructure changes as well. And you probably want some dynamic configuration to avoid a redeploy just because some backend IP changes. But by keeping possibility for change small, having clear definitions of what can and what can't change, what is state that is supposed to change during runtime as opposed to state frozen at built time, it reduces the cognitive overhead required to understand an infrastructure at a given point in time.

### Container vs VMs
Some might argue that all this is possible with VMs already and conceptually this is entirely true. Although in practice the overhead imposed by virtualization make it less feasible to run every little service in their own VM. And once you start making exceptions, you need to manage those exceptions, so you need to account again for mutable systems and raw applications with all their knobs.
