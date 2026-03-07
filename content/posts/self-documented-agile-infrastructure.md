---
title: "Self Documented Agile Infrastructure"
date: 2009-03-02T14:13:00Z
draft: false
categories: ["Technology"]
tags: []
---

In my latest position, as an IT Operations Manager I was confronted to the classic problems of a non-mature Operations: We were understaffed, in a fire-fighting mode, there was poor documentation (either missing or not up-to-date, often misleading), almost no backup, and the team members had almost no overlap in their skillsets and were demotivated.

I couldn't afford to lose a single person of my team as the knowledge lost would be dire for the company, and to make things even more complicated, our CEO wanted us to be able to deploy our home made software to remote client sites.

On the good side, one of my team member had an excellent knowledge of the home made software, another was a good perl developer, there was a good knowledge of Suse, rpm packaging and they already had a set up a subversion repository and a basic puppet setup.

To consolidate the knowledge and move away from manual operations, it was decided to use svn, puppet, Suse and pxe to build a self-documented agile infrastructure where anyone would be able to deploy new services.

## The basic blocks

The applications was packaged using rpm and the latest valid version stored on a file server, but all the configuration files (including those needed to build the packages) were stored in subversion.

This way, it was possible to keep track of the changes (who, why) while at the same time having a way to retrieve the latest valid version using a simple 'svn co'. The svn commits were sent to all team members, so it kept everyone informed of what was going on.

## The recipes

The services and server setup were described in puppet and stored in subversion. The services were described in a generic manner using templates as configuration files so you could instantiate a new service by deploying the needed rpms and creating "on the fly" the configuration files adapted to that specific instance. The important idea was that no manual operation was needed to deploy a new service thus allowing it to be perfectly reproductible.

Thanks to this solution, one could easily deploy a new instance of a service on either a physical or virtual machine. As we were in a j2ee world with a multi-tiered application, you could either stack several services on a machine (for development or testing for instance) or one service per machine, depending on your needs.

The nice side effect is that puppet is the live documentation of your systems as it defines and enforces the active configurations! Since the puppet files are also stored in svn, it is possible to see all the changes for a file through time with the associated comments.

The drawback of the system is that extreme care must be taken not to manually tamper with the configuration of the servers: everything MUST go through puppet, and the comments must be kept relevant.

## The deployment system

The machines could be either physical or virtual machines, and pxe combined with kickstart is used to deploy a basic setup consisting of a basic Suse + puppet. Of course the kickstart files are stored in svn. Once the server is deployed, puppet can then populate the server with a set of services/configuration.

## The backup server

Since a service/server could be easily reinstalled using this solution, there was no need to backup them which is a big time and tape saver.

This way you can concentrate on saving your application data, that is your production dataset as well as the files on the file server and the subversion repository.

In our setup, it was decided to sync the subversion repository and the files stored on the fileserver between 2 sites. Also, thanks to the use of subversion, everyone in the team had the files on their own machine.

## Disaster recovery

During the implementation, cross-dependencies between the subversion, installation, puppet, file and backup servers were considered in order to allow a complete restoration of the infrastructure, provided that we had access to the backup tapes and could reinstall the backup server manually using a Suse install media.

It was decided that the subversion, file, build and installation services would be installed on a single machine. From there, you could reinstall the puppet server via a very limited set of operations that were documented with care (basically, installing the packages and checking out the svn repository).

Once this is done, and provided all your infrastructure is described using puppet recipes, you can easily repopulate your servers in a case of disaster recovery, but it could also be used to install everything on a remote site, provided you have a machine were you can bootstrap your infrastructure.
