---
layout: post
title: Designing a terraform workflow - part 1 (the context)
---

For my current client, I had to design and implement a green field solution to manage AWS resources.

Because it is a company operating in a regulated environment, there are a few regulatory constraints that need to be met, but the idea was to automate them as much as possible as part of the workflow instead of having manual steps and policies.

A very interesting aspect was that my client wanted to allow as many of its employees as possible to experiment with new AWS technologies as freely as possible, instead of relying on a limited team of "devops" designing "blueprints". The idea was that leveraging AWS innovation this way would be more effective. But at the same time, there was a need to have safe-guards and controls in place to meet the regulatory requirements.

The main constraint for the workflow were:

- reproduceable, auditable, testable, self-documented
- allow parallel changes and branch builds
- allow people outside of our team to contribute so we wouldn't be a bottleneck
- have multiple environments (test, production) managed by the same code
- detect resources dependencies that would make bootstrapping the whole set of ressources from scratch an issue (for Disaster Recovery for instance)

### Why terraform?

When looking at managing AWS resources, the default choice is obviously CloudFormation.

However I decided Terraform was the right tool for the job in that case because of a the following.

#### a better format

I believe that the terraform DSL is a better choice than the pure JSON format of CloudFormation if only for its native support of comments, but also because it feels a bit terser and higher-level.

The alternative in CloudFormation if you want comments (without using hacks like metadata) is to use another layer of DSL or tooling to generate the JSON files.

The ability of Terraform to support arbitrary segmentation between files in the same directory makes it easier to maintain in my opinion and also allows for longer files. The limits in CloudFormations can be pretty drastic when you have a lot of resources.

The module support in Terraform is also much better IMHO than the CloudFormation nested-stack approach to promote reusability (nested stacks counts in the stack limits, and are hard to debug when they fail as the logs are not accessible from the console).

#### resource types

While terraform is a bit behind in terms of resources supported when compared to cloudformation, it is catching up quickly and I believe that will only accelerate: by its opensource nature, there are increasingly more and more devs behind it, while on the other end the AWS CloudFormation team is a finite resource (and pretty small too I gather, unless AWS decides to ask for the feature teams to do the CloudFormation integration too).

But the main difference here is while it is possible to create custom resources in CloudFormation, once they are natively implemented, you need to destroy/recreate them, which might be impractical. In Terraform on the other end you can create manually resources and then "reclaim" management of existing resources once the native support is added in terraform, by adding the corresponding information in the DSL and the state files.

#### ownership of the management and orchestration layer

While there are SaaS alternatives that could have provided an easier way to manage resources through a Web UI for instance, it was important for my client to keep a control and ownership of that management layer because it is operating in a regulated environment.

This also means that something like Atlas was not an acceptable solution so we would have to reimplement things that Atlas would provide, such as locking/concurrent changes.

#### multi-cloud

Probably the most obvious: while the initial implementation for the project is in AWS, my client wanted to keep its options open and be able to use the same tool to manage other cloud resources (public, private).

#### Open-source, free and extensible

The open-source nature of terraform means that the client would still be able to use it for years to come, even in the eventuality of a supporting vendor disappearing.

It would also allow for code auditing if required.

Its open-source nature and extensible design means that the client would be able to add the missing AWS resources required (some work is on the way for instance to implement [EMR support](https://github.com/hashicorp/terraform/pull/6492)) or adapt it to internal/bespoke solutions if required.
