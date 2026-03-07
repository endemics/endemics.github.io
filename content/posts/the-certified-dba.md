---
title: "The Certified DBA"
date: 2010-03-25T02:11:18Z
draft: false
tags: []
---

Yesterday my friend and ex-colleague [Iain](http://iain.cx/) send me a link to this [dailyWTF article](http://thedailywtf.com/Comments/The-Certified-DBA.aspx). I felt the content of the article and of (most of) the comments were so wrong on so many levels I had to write something about it...

## The RedHat Performance Tuning course

I suspect he did this because he remembered a heated discussion I had in a team meeting with our team leader back when Iain and myself worked together: our team leader was coming back from a "Red Hat Performance Tuning" course and said there was a lot of things that we could do to improve the performance of our systems, including:

- ensure that all systems had swap defined as twice the amount of RAM
- ensure that the /tmp partitions were created on the outside parts of the "spindles"

I expressed serious doubts about the validity of those assumptions in a modern IT environment.

First of all, memory is cheap nowadays and QoS matters. In most cases, a swapping server is the best way to guarantee that it won't be able to offer the right level of service: it is either an indication that there is something wrong with the software like a memory leak, or that the server is not properly sized for the task.

The partitioning issue is very similar to the case described in the dailyWTF article. It is based on physical assumptions that are not necessarily true nowadays, especially when we are talking about partitions made on a hardware raid1 volume using multi-platter drives. In my opinion, there was no guarantee that the firmware of the RAID controller nor the one from the drives will do what we think they do.

## Proof versus Belief

Interestingly enough, it seems that I was wrong and that drives manufacturers do their best to keep a mapping that is still in sync with the belief system in place, as the proved by the [zcav tests](http://www.coker.com.au/bonnie++/zcav/results.html) pointed in one of the article's comment.

What is important here is the experimental evidence as opposed to beliefs or possibly outdated knowledge.

Still, it is important to remember that the zcav published datas are only valid in the context of the tests: they might not be valid for your production system with your set of drives, your raid controller and moreover, for your application needs.

Of course, with enough experience with a specific application, there is a possibility that generic rules can be deducted, as long as they are methodically deduced rules and not just wild assertions.

Alas, if the conditions changes, the rules are no longer valid, so you can't blindly follow them when you do performance optimization, you can just use them as hints or possible things to try: only in-situ measures can validate an hypothesis and prove performance increases.

Which means that if you have to optimize performance, you have to use your brains, common sense and produce reproducible test results!

## Overcomplicated setup

With such a complicated setup, it can be difficult to measure the right thing and there can be plenty of unwanted interactions.

On the other hand, if you can't prove it makes a difference, you just over-complicate for the sake of it, which means it will be more difficult to maintain and diagnose for no provable benefit.

If it brings more performance, the benefits will still have to be evaluated against the operational risks that the complexity brought.

Not only the amount of complexity but also where you add the complexity matters: the more complexity you push down the stack, the harder it is to change things: a configuration option in an application is easier to do (and revert) than changing the version of the software.

If you depend on a specific software and hardware stack for your system/application to work, you are tied and have very limited ways to make your solution evolve or adapt. This tend to create systems where changes induce more risks.

It might not necessarily be a problem, especially if your system does not evolve a lot either in functionality or scale, and the risks can be mitigated by tests, but those costs must be clearly understood when the decision is taken.

Usually a lot of optimization can be done on the highest level, i-e the user side, with limited risks and efforts. However it can't be achieved if you don't understand what you're doing nor if you don't understand what the client application/users are doing,

Instead of shooting in the dark by applying random recipes, talking to the users to get the Big Picture can help making the system more in sync with the actual needs, and will let you identify which path that can be explored.

Sometimes it can be as easy as spreading the load over the course of a day/week/month instead of having everyone doing their queries at the same time.

Also, provided the DB is not used by a blackbox system, there are different things that can be done either on the DB or at the application system:

- pruning the tables
- optimizing the schema/indexes/queries
- queuing/asynchronous queries
- spliting/sharding the tables

From my experience, the need to do performance tuning usually comes from a bad design and the inability to scale horizontally. The IT industry has been relying too much on the ability to do vertical scaling, and unfortunately, it seems that apart from the big web players, only a few companies have realized that vertical scaling is barely an option now.

## The communication problem

I believe the communication issue and lack of trust to be the most fundamental problem.

It is obvious that the company has a "Us vs Them" syndrome between DBAs and Ops, indicating a big silo problem, and I doubt this problem is only limited to the [Ops DBA interaction](http://www.krisbuytaert.be/blog/devops-secops-dbaops-netops), but probably spans to other teams interactions as well.

I think the Ops persons and his boss did the wrong thing there by hiding things under the rug. I believe that it was only pride that made the Ops guy behave the way he did, and I think this will only create more problems for him in the future.

Maybe a better way to do is to show the DBA that there was a better way to do things on a system level, to create a trust relationship with him and to encourage communication with the DB users.

Pouring oil on the fire is not going to stop the fire spreading nor the false beliefs...
