OpenStack Software Deployment Manifesto
=======================================

Ursula exists for a few reasons. First and foremost, its job is to deploy and configure an OpenStack cloud. This cloud is made up of many components, some required, some optional. At the heart of the cloud is the OpenStack software set, and beyond that is the tools and services that are required for OpenStack to complete its job. Ursula deploys and configured each of these components to present a functional cloud. Additionally, Ursula can be used to upgrade components of the cloud, either the OpenStack code or the other ancillary parts.

While Ursula is well suited for deploying a single cloud, things get interesting when the desire is to deploy multiple clouds, or to deploy a cloud multiple times and expect the same results in every deployment. Historically, this expectation has not been met, although much work has been put in to make strides toward this goal.

This document exists to outline the set of problems Ursula faces, the work that has been done thus far to mitigate those problems, and to hypothesize a potential solution that moves even further toward resolving the stated problems. The intent is to work from the problem set, rather than to introduce a solution and then seek out the problems it solves.

The Problems
=============

## Python package requirements
OpenStack is made up of software, a LOT of software. This software is primarily written in Python and depends greatly on a variety of Python packages that are not themselves maintained and distributed by OpenStack. As such, to install OpenStack, one needs to gather all the requirements, or dependencies, in order to run the various parts of OpenStack.

In general, these dependencies are not explicitly perscriptive in the versions of software required. When Ursula was first being developed many of the requirements listed by the OpenStack software set were open-ended, just the name of a python package and no version information. While this method may work for the single installation of a single OpenStack cloud, it does not work well for repeated installations over time.

When left to its own devices, the classic methods Python uses to gather the requirements will result in installing the "latest" available releases for each requirement (and the transitive requirements thereof). What this means in plain terms is that if you install OpenStack today, you'll get a set of python packages of certain versions. If you repeat the installation process at some later time, *without changing any of the inputs*, the end result set of python packages has high likelyhood of being different. Not only does this lead to a different software set, it often leads to a *broken* software set, as combining OpenStack code from the past and python packages from the present creates an untested combination.

This potential for change over time obviously poses a challenge to creating reusable and repeatable installation methods. Fortunately steps have been made, both upstream within the OpenStack project and downstream within Ursula, to mitigate this potential for change. The following solutions outline the steps that have been made thus far.

### Constraints
The OpenStack development community realized the importance of providing a more perscriptive set of requirements. To that end, they began producing a set of constraints, a listing of python packages with explicit versions that were present when a given set of OpenStack code passed through the various quality gates.

What this constraints file provides is the specific version for every requirement across the entire set of co-gated projects, such as Nova, Neutron, Glance etc. With this file, a deployer can combine a specific version of Nova with the matching version of the constraints file and be able to reproduce with certainty the same end result combination that was used in the gate tests. Not only that, this installation method can be reproduced over and over again over time without resulting in change.

Recently, Ursula begain making use of this constraints file when it installs the OpenStack projects, and this had greatly increased our stability and lowered the number of times when our own quality gates failed due to version changes outside of our own.

### Packaging of OpenStack
Prior to Constraints being introduced, another solution employed by Ursula was to consume pre-built packages of OpenStack software. This removed the need to build the OpenStack software on every host for every deployment. The OpenStack code set would be built once and utilized over and over as an artifact.

At Blue Box, the packages consumed by Ursula are produced by Giftwrap. Giftwrap provides an easy process to construct the packages of OpenStack software and its dependencies and solves many of the problems associated with packaging software, but that's a story for a different document.

A recent enhancement to Giftwrap is to make use of the constraints file mentioned previously, so that the content in the package matches the content that was used during upstream testing efforts.

## Non-OpenStack Packaging
Similar to the problems with OpenStack dependency versioning, there exists a problem with non-openstack code, such as RabbitMQ, Percona SQL, Ruby gems to support Sensu, and so on. When these packages of software are installed, either from an Operating System vendor or from a software vendor, or from a language package aggregator, care needs to be taken about which versions, if any, are requested. However, even with careful description of every version, the transitive dependency resolution routines can still result in a difference in all the aggregate of the versions.

So far, little has been done to resolve this problem, other than being more perscriptive about first-level versions of software required to run an OpenStack cloud.

## Deployment Length
Another high level concern consumers of Ursula have is the length of time it takes to complete a deployment from start to finish. While there are many contributing factors to the length of a deployment, one of the biggest contributors is the length of time spent installing software. This problem is compounded when you consider that the same exact actions are taking place across every node in an OpenStack cloud. In addition, every deployment run, even if the only change is configuration files, the same tasks are repeated to ensure that the desired software is in place. This is a large amount of wasted time.

A few partial solutions have been implemented to alleviate the length of deployment, but so far a comprehensive solution has not been proposed.

### Ansible Tagging
In Ansible, the technology behind Ursula, there exists the concept of tags. Tags allow a developer to mark sub-sets of an overall set of tasks in descriptive ways. Once marked, deployers can explicitly target these sub-sets of tasks to avoid needlessly executing the entirety of the task set. This can provide "short cuts" when deploying a known small set of changes to a particular component of the OpenStack cloud.

In addition to being able to target sub-sets, tags allow a deployer to *skip* sub-sets of tasks as well, which is handy if it is know ahead of time that a one or more sub-sets of tasks will not result in any meaningful change.

## Unrealiable Quality Testing of a Release
This problem is related to the fact that time passed can change the outcome of an OpenStack deployment, even if none of the inputs to that deployment have changed. When Blue Box prepares a release of it's OpenStack product, vigorous testing is performed to ensure not only the ability to perform the deployment as described by documentation, but for that OpenStack deployment to perform in a way that meets high quality standards. Unfortunately, due to the fact that software content within an OpenStack deployment can differe from day to day, the assurances provided at release time cannot be relied upon as time passes.

### Shorter release cycles
One way to mitigate the drift of quality over time is to reduce the amount of time any one release is deployed. If new versions are produced and tested frequently, and deployed frequently, then the amount of time a single release will "live" can be relatively short. This reduces the chance for change.

Unfortunately this strategy merely works around the problem rather than solves it, and faces increased pressure from downstream teams and customers as time between deploys gets shorter and shorter, as each deploy still creates some form of instability or interruption to the cloud itself and costs time from the engineer performing the deployments.

## Unreliable external resources
Because OpenStack and the ancillary services requires many peices of software to be installed, there are dependencies on external systems, such as package repositories, pypi repositories, gem repositories, git repositories, file repositories, etc... Each of these resources representes a potential point of failure during a deployment.

### Mirrors
At Blue Box we've introduced mirrors that we control to provide the external resources we require during a deployment. This greatly reduces the risk of failure during deployments, but it does not eliminate it all together.

## Software Auditing
A less thought about problem is the desire to perform audits of the software that makes up an OpenStack cloud. These audits have multiple purposes: software licensing compliance, security vulnerability assessment, version drift from previous deployment, and more. While software auditing itself is not a problem, when combined with software sets that can differ over time it becomes difficult to expect the results of auditing one OpenStack cloud deployment to match the results of another OpenStack cloud that was deployed using the same inputs, but at a different date.

Ideally auditing could happen at "build" time and be relied upon for any environment that was deployed from that "build". Unfortunately the way Ursula exists today means that the "build" time is in fact the deploy time for many parts of the software set.

### Giftwrap metadata
When using Giftwrap to build packages for OpenStack software, a plugin is available that will audit all the python package versions, and their associated licenses, at build time. The results of this audit can be relied upon for any OpenStack deployment that utilizes this particular set of Giftwrap build packages.

A possible solution to many of the problems
===========================================

Now that the problems as they exist have been outlined, a theme to the problems is made clear. Drift over time of a static process is the enemy, and thus is the key problem to address.

The way to eliminate drift over time is to eliminate the "build" steps of an OpenStack deployment, so that the static process of using a set of inputs to deploy an OpenStack cloud result in the same set of software result that is reliable and repeatable over time.

a potential solution to these problems can be discussed. This potential solution is the containerization of the OpenStack and associated software required to implement an OpenStack Cloud.

The concept of containerization is very broad, and can quickly get complicated. Instead of "going down the rabbit hole" of complexity, the imagined approach here focuses on solving the above problems with as little added complexity as possible. What containers brings to the table for Ursula is the ability to encapsulate a collection of software in a light weight relocatable artifact that can be used over and over again. In essence, it allows us to "build once" and "deploy often".

## The Approach
The main goal of containerization for Ursula is to "build" all of the things we need for our deployment once, instead of repeatedly on every system for every deployment. This includes the OpenStack software and the ancillary software required by the OpenStack services. In an ideal world, all the software we deploy would be containered, and the only thing running native on the target host is the tooling to deal with containers.

Building once also addresses a number of the other problems listed above:

### Version Drift
By building once we eliminate the potential for version drift. The build phase for any given release happens only once and the results of that build are used over and over again, so the versions cannot change from one day to the next.

### Software Auditing
A single build step provides us a place to perform an audit of the software and rely on that audit being accurate over time.

### Non-OpenStack Packaging
By building exactly once for a given release, the versions of non-openstack packages do not have an opportunity to change. This will resolve the vast majority of issues with non-OpenStack packaging.

### Deployment Length
Building in place for every deployment is costly. When this action is done once and the output is re-used every time, this cuts out a significant amount of the deploy time. In addition, some container systems allow for transporting deltas for changed containers, which can reduce the amount of content transferred on update of a given system. A deployment would then be simply ensuring the correct container images are on the system, laying down configuration file changes, and restarting the appropriate services. A much smaller body of work.

### Unrealiable Quality Testing of a Release
If the build step is removed from the deployment, then there is little chance for the outcome of a deployment to change over time, so there is little chance of the Quality Testing becoming increasingly invalid over time.

### Unreliable External Resources
While using containers does not eliminate external resources, it can eliminate the use of external resources during a deployment. The use of external resources can be limited to just the build phase. The only resource required during a deployment would be the resource that houses the container images.

### Software Auditing
Software Auditing becomes more reliable as the build phase, where versions can change, has been eliminated from the deployment. Full auditing can be done at the container build phase.

## Impelementation
There are many things to consider when going down the path of containers. As such, only vague implementation details are provided at this time, as some discovery work is required to learn all the details that will need to be worked out.

The broad approach is to make an iterative change to the way we deploy today, to target the problems outlined in this document. We are not interested in a complete redesign of our deployment method.

### The build
In simple terms, our approach will be to build containers for each major OpenStack system, the clients, and each ancillary service we require. These containers will be just the software, none of the configuration. 

### Services
We will write out init system scripts for each of the services we need to run, which will in turn start / stop / restart the container image for that service. 

### Configuration files
On the host operating system, we will continue to write out configuration files as we do today. The configuration files for the service will be mapped into the container as a volume, so that the runtime configuration is made available to the software within the container.

### Networking
Much needs to be investigated here, but the desire would be to not complicate things as part of an MVP. We could simply use host networking inside the container, allowing services to bind to ports as they always have. We do not currently allow multiple versions of a service to run on a given system at the same time, adn we wouldn't allow it with containers either.

### Privileges
Many services will not require "root" level privileges. When possible will only provide enhanced privileges to those services that require it.

### Gotchas
???
