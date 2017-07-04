---
title: 'Scope and Ownership in Tech Companies'
date: 2015-04-22 12:19:36 +0000
tags: 
layout: post
---
![ownership](/content/images/2015/Apr/acient_jobs.jpg)
My first job was a typical corporate job at a financial service provider after which I worked at large hosting company before joining SoundCloud, a startup culture-wise closer to Silicon Valley startups before joining Docker end of 2013.

Although I never was a manager, I was always interested in the company as a whole, how my work shapes it and how the environment impacts my work and how other companies are organized.

On thing that strikes me most is that not only that we lot of companies can't agree on *who should do what* but that lot of companies don't even think about this carefully and don't realize that this affects everything else. They strives for a Microservice architecture without even knowing why and name their Ops team "DevOps" because that's what people do.
I want to provide my thoughts on questions around Scope and Ownership but leave the implementation details to the reader. I'm happy to answer technical questions in the comments though and might blog about those implementation details eventually if interested.

# What your Company should do
[![Scope of Company](/content/images/2015/Apr/3355654186_6751a40056_z.jpg)](https://www.flickr.com/photos/jo-ghadban/3355654186/in/photostream/)
No company does everything on their own. You don't build your own servers, let alone CPUs and chances are high, you don't even run servers on your own.
All this is fine. It's not your business to build CPUs, so you shouldn't. Same for managing datacenters and servers. If it's not your core business, you probably won't ever be as good at it as someone specialized in it.

On the other hand, never outsource what makes up your company. You can't have someone else build something crucial and tightly related to your companies overall business. If you don't do your own Community Management, Support or PR, people will realize that your company and brand has nothing to do with your social media presence.
Outsourcing this is like outsourcing your face.

If your product needs custom software, you should build it. It still seems like a lot of companies believe software can be bought in like car service parts without realizing software is never done. It's not even ever bug free, so it needs constant attention by people who feel part of the company and see purpose in what they are doing.

Software is also usually the interface with your customers and you don't want something you don't *own* between you and your customers. Not only that you don't understand and can't control, but all feedback loops are much wider. You listen to your customers, explain your ideas to some agency, they explain it to their developers and after some time you get a result. For every problem with that piece of software, you need to go through the whole loop again. Every competitor doing this in-house will move *much* faster due to their short feedback loops. You see a problem, you fix it.

Even internally, think about what makes up your company and how much of the internal infrastructure and software you want to own. If you are a small web shop, you're good to *use* existing frameworks and services. You can usually expect them to work for your use cases. If not, you need to fill tickets and wait for someone to fix them.
Now if you are company operating at large scale, you need to actually *own* those services since you will run into issues nobody else ran into. In our open source world, this doesn't necessary mean to write your own software but have people to *own* those components, people able to understand their internals and able to fix bugs without any vendor supporting them. You see a problem you fix it.

# What your teams should do
![Team](/content/images/2015/Apr/samurai.jpg)
No matter how you build teams or how you call them, I believe they should be as independent from each other as possible while providing benefits to all other teams where possible.
As too many cooks spoil the broth, having many engineers working on the same code base usually isn't working. You run into all kind of problems, from merge conflicts to bikeshedding and unclear escalation paths.

The probably best way to avoid this is by splitting your monolithic application into (micro)services. Each teams develops their service independently. Since services depend on each other, you need to make your APIs backward compatible or introduce some kind of versioning.

## You build it, you run it
Each team does not only develop their own service, they also deploy, operate and monitor it. You will never fully understand how an application behaves if you don't run it on your own. With the right monitoring in place, it's usually not that hard to isolate in which service's domain a problem occurred. This guarantees short feedback loops and sets the right incentives to balance features and work on technical debt.
This also implies that there is *no ops team*. Ops isn't a team, it's a role, something everyone should be doing.

## Self-service
The teams being able to own each aspect of their service requires a self-service infrastructure.
It depends on how similar the requirements of different teams are and how much operational experience they have.
If you're running on AWS, you might just give every team their own login and ability to spin up machines and resources as they need. But this requires quite some operational experience to make this secure and reliable and there will be lot of pieces that every team needs to re-invent.
Therefore, the self-service platform should address all needs that are similar across all your services and support. It might look like Heroku or Elastic Beanstalk but might be more opinionated and specific to your company.
Those infrastructure services are the same as user facing services. Some team builds and operates them, it's simply a service for which customers are other engineers within your company.
This isn't Ops and it's not DevOps either, although due to the nature of the service it might involve more systems knowledge than frontend development.

## Scaling beyond that
At large scale funny things happen and in a large company, you want people specialized in those problem. If you have less than, let say, 200 engineers, you probably don't.
Those scale issues are similar across services but nothing you can easily abstract away. To solve those problems, you need to look at the big picture. The problem domain is different which is why companies like Google and facebook introduced a new role. Whether you call it *Site Reliability Engineering* or just *Production Engineering*, please don't call it Ops and don't use it synonymous with. It's not about *operating* anything.

# Conclusion
Some things proposed here might not be trivial to implement. It's hard to hire people with operational and development skills and a Microservice architecture opens up a whole new class of problems in the realm of distributed systems, but in the end most successfully companies seem to arrive at similar conclusions. If you disagree and choose a completely different approach, that's fine as long as you do it deliberately.
Nothing is worse than having some specific form of organization without knowing why.
