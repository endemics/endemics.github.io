---
title: "On the Shortcomings of Systems and Networks Engineers Training"
date: 2009-01-16T20:29:00Z
draft: false
categories: ["Technology"]
tags: []
---

As far I know, there is no course to become a Systems and Networks Engineer, aside from courses to learn (and gain certification in) a given vendor's product. In fact, back in my university years, I remember that my teachers seemed to assume that there was no interest in this kind of thing as learning the options and caveats of a particular product was all you needed. In their eyes, algorithmic and development approaches (RAD and OO at the time) were where the real focus lay.

In my case, the situation might have been worsened by the traditional friction in France between university (were the "real, pure, academic" research is done) and the Ecoles d'Ingénieur (where you learn about engineering and sometimes conduct "applied research"), but I'm not so sure the situation would have been so different in an engineering school or another country (I'll be interested in your feedback there to prove me wrong!).

So, how does one becomes a Systems and Networks Engineer? Well, it's easy, you learn by yourself, usually starting with a small set of machines and mainly by a trial-and-error approach. If you're lucky enough, you might benefit from someone else's experience and coaching. But still, it remains mostly an ad-hoc approach.

Of course, you quickly learn to avoid tinkering with the production platform on a Friday evening, and given enough experience you can even begin to "guesstimate" - to a greater or lesser degree of accuracy - the impact of such-and-such a modification, then hopefully the number of systems you manage will increase until eventually you find out the hard way that complexity doesn't grow linearly with the number of systems.

I would even claim that given the chance to work with different environments and large scale platforms (highly available, highly loaded web platforms; HPC clusters; heterogeneous banking environments), one might infer common rules of thumb and even have the hubris to try to find a meaning in the chaos.

The fact, however, is that I believe this ad-hoc approach to learning the job and the lack of (field proven) best-practice references to be The Source Of All Evil.

First of all, from this learning process comes an approach comprising unproven beliefs, mythology or carved-in-stone rules ("one needs twice the amount of ram as swap space"). It also makes it difficult to assess someone's ability as a Systems and Networks Engineer if not by considering her technical knowledge/certifications or previous experience in a similar position.

Secondly, the good practice of "not changing what works" forged by the trial-and-error approach, tends to encourage cruft accumulation and creates a certain reluctance to change anything at all. As a result risk-mitigation approaches such as continuous integration and minor steps are replaced by "big-bang" style changes with increased risks of failures.

All in all, I believe that it has created a situation whereby IT Operations is working against the (in my eyes desirable) goal of becoming agile and business-oriented - a true competition differentiator and not just a "cost center" working in firefighting mode.

The "cost center" aspect has motivated the few approaches trying to address the lack of maturity in IT Operations: ITIL, Cobit and so on. To the best of my knowledge, they are all process-oriented and mostly address the problem from a financial perspective (ROI, risk management).

While I believe there are interesting ideas in all of them, and that cost is an important factor in the need - solution equation, I am not too convinced by the "process" approach which limits risk but adds weight and inertia to the organisation and kills pleasure and innovation. I confess I might be too influenced by the ideas of the [Agile Manifesto](http://agilemanifesto.org/) here, but I can't stop myself thinking that neither Google nor Facebook used ITIL to get where they are.

I also find them too complicated to be real enablers and believe that even though they warn against it, they incite dogmatism where pragmatism should rule. Because of this, I think they fight against the exact goals they are trying to achieve.

So how can we get out of this mess?

We would definitely benefit from an increase in interest from the academic world towards IT Operations and Infrastructure realities. Consider Google's study on [Hard Drives failures](http://research.google.com/archive/disk_failures.pdf). Before its publication different people had wildly differing beliefs about disk failures based on factors such as: their own experience with a statistically-insignificant sample size of drives; manufacturer advertising (propaganda); luck. With a large scale, scientific study to turn to, people gained a much better understanding of the subject matter.

Naturally, courses about availability, scalability, large scale systems and networks design and management would be welcome in Universities.

But successful companies such as Google or Amazon couldn't have emerged without good IT engineering practices and a sound infrastructure (after all Amazon even sells its services now via EC2 and S3!), so, it is certainly possible today to build an IT infrastructure that makes a difference.

Then we definitely have the responsibility to learn from those leaders and spread that information around if we want IT Operations and Infrastructures to mature and serve the business and our own users (kudos here to websites such as [High Scalability](http://highscalability.com/) or [Storage Mojo](http://www.storagemojo.com/) for their excellent work).

Undoubtedly most of the technologies those companies use to manage their infrastructures are purpose-built in-house developments that won't be published, so we as a community need to build the tools we need in the same way developers have started open-source re-implementations of well known building blocks such as MapReduce for instance [hadoop](http://hadoop.apache.org/core/).

Tools such as [Luke Kanies' Puppet configuration management](http://madstop.com/), rapid deployments tools such as [openqrm](http://www.openqrm.org/) or easily adaptable and scalable monitoring systems such as [hobbit (now renamed Xymon)](http://hobbitmon.sourceforge.net/) should be endemic to our infrastructures, yet they are sadly too often an exception.
