---
layout: post
permalink: /blog/container-rootfilesystems-in-prod
title: "Container Root Filesystems in Production"
date: "Sept 08, 2017"
---

This post is a loose transcript of a talk a colleague and I gave at [Container Camp UK 2017](https://www.youtube.com/watch?v=lctMC1WNd1U&t=74s), in
which we discussed our team's use of container root filesystems in production.

[Tiago Scolari](https://github.com/tscolari) used to work with me at Pivotal, and in 2016 he and I began work on a new project
dedicated to providing the backing root filesystem for Cloud Foundry containers. For this talk we went over
how we came to work with the filesystems that Cloud Foundry containers use today.

Here is a summary of what this post will cover:
- glossary of terms for those who have less familiarity with Linux filesystems.
- a quick overview of Cloud Foundry, describing specifically the application runtime components: which bits run your app in the cloud and how CF does that successfully.
- each implementation and filesystem which has been used in Cloud Foundry containers since the project began.
  For each I will explain why it was chosen, how the team worked with it or around it, and why
  we decided to abandon it for another.

## What is a filesystem?
There are a couple of meanings associated with this term.
The first is that is it is the directory hierarchy used to organise files on a system. On Linux the
start of this tree is `/` (or root) in which there are subdirectories and then further subdirectories and so on.
Another meaning is the type of filesystem: How the storage of data is organized on disk within that hierarchy.

Different types of filesystem have their own set of rules for controlling the allocation of disk space to files and for associating each file’s metadata.
For large scale systems, making the right filesystem choice, for both the host and the containers running on it,
is very important: the wrong filesystem type can have a very noticeable effect on performance.
Each one brings its own set of tools and tricks for managing data, and these can react in sometimes
unpredictable ways under load; being prepared to face the risks and ready to adapt is key.

## What is a _root_ filesystem?
A rootfs is the filesystem contained on the same disk in which the root directory is located.
It is onto this FS that all other filesystems are logically attached, or mounted, as needed at startup.
The rootfs contents and directory structure (first meaning) will vary with each OS. The type of filesystem
running and controlling the allocation of data of this rootfs hierarchy will vary depending on how the system was provisioned.
CF needs to provide both the correct rootfs structure and the correct filesystem type to manage data within
that structure for every container. So when we talk about the rootfs used to back application containers,
we mean both, but the main focus in this talk will be the mechanism (fs type) by which CF controls data and manages resources.

## What is Cloud Foundry?
Cloud Foundry (CF) is an open source Platform as a Service and was first released in 2011. Nowadays it is very popular with enterprise
and banks etc.
It’s a big micro-service project with dozens of repositories, mostly written in Go with some Ruby lurking here and there.
Today CF supports both buildpack and Docker image based apps, and will run each application in a container with full isolation one from another. 

### Buildpack?
But the first versions of CF only supported buildpack applications. So what is a buildpack?
A buildpack is a set of scripts that provide framework and runtime support for application code, they are
language specific and are used when you push a new app, or update the code of an existing app.
Officially supported languages right now are:
- binary
- go
- java
- .net
- node.js
- php
- python
- ruby
- static files

Everytime you push new code, CF will create a container from a known rootfs structure based on a custom version of busybox.
Cloud Foundry will then copy will copy the buildpack and the application code into that container. Next, the buildpack will execute and
compile the application producing a droplet.

If you are deploying a Ruby application, the droplet will contain the required gems. In the case of a golang one, it will have the compiled binary.
Each buildpack will handle compilation and dependencies in the way that makes sense for the language.
The droplet should therefore have everything the application needs to run, apart from the root filesystem which is provided inside the container.
This allows us to scale and move applications quickly and even patch security fixes into the rootfs without the need to recompile apps at all, as the
code can simply be moved to a new container, with a new rootfs.

![alt text](/assets/images/buildpacks "buildpack application")


### What does CF need from its containers?
Cloud Foundry was designed from the start to run as a multi-tenanted platform. This meant that many different organizations and independent developers
would push their code to the same platform. To do this securely, and to provide a good user experience for everyone, their applications should run
in isolation; they should neither interfere or be interfered with by others.

As well as stopping interference between applications, the applications should also not be able to interfere with the host's processes or filesystem.
The simplest way to ruin the files for someone else is by writing so much data, that nobody else is able to. For this reason CF enforces disk
quotas on their applications.

In this post I will stick to keeping apps away from the host fs, but please check out my [other post](/blog/the-route-to-rootless-containers) to learn more about process isolation
and security.

## Containers? In 2011?
At the time, the kernel which CF components were running on was 2.6. This meant that most of the things which make containers the things we know and
love today (things like the user namespace), just weren't there.

## "Warden"
Back then, Cloud Foundry’s first “container” implementation was Warden and Warden used AUFS to manage its container root filesystems.
We didn’t yet have a nice concise word like “containers” for these “isolated resource-controlled environments”, but
containerisation technology was already in play long before Docker hit its stride. Warden was the platform’s first simple API,
written in ruby and C, and managed these environments as a daemon. Warden initially used the LXC (linux containers project)
which tied it to Linux. Since Warden relied on a small subset of LXC functionality, this tooling was then dropped with Warden implementing only what it needed.

But that is containerisation in general, we’re here for the filesystems.

### Why AUFS?
AUFS is a filesystem type which originally stood for Another Union Filesystem but since version 2 has stood for
Advanced Multi-layered Unification Filesystem. AUFS was developed in 2008 and is a complete rewrite of the Union
FS aiming to improve performance and reliability. Like UnionFS, it implements a union mount to combine several directories
to make their contents appear to be under just one directory.

At the time Warden began, there were very few filesystems mature enough to be used in the way CF needed.
At this point every container root filesystem’s structure and base contents were identical, so mounting each
one through AUFS was far more resource efficient than just using copy.

But AUFS was not available in the mainline kernel: it had to be patched in from an external module which lead to a lot of maintenance overhead.
Furthermore AUFS had no native way to enforce user disk quotas, which was essential for a multi-tenanted platform.
So the original team did what it could to make it work.

Here is a diagram which gives a general idea for how Warden made CF’s requirements of scalability and user disk quotas work.

![alt text](/assets/images/aufs "aufs mounts")

The hosts on which application containers ran were deployed with a tarred linux root filesystem. When an app is pushed,
this rootfs is untarred and its contents serve as the base read-only layer for each container. Warden then mounted each
application instance from this base using AUFS. The resulting thin read-write layer became the isolated root filesystem
for the container. The application code was placed into this layer and the app could run and make changes to it’s own r/w
layer of the rootfs, without affecting the base-layer or, of course, the host.

As for CF’s quota needs, application disk quotas were enforced by UID: each application container was run as a different
user on the host with each user having its own resource limits enforced by the system quota control.

But of course things are always changing. In 2013-2014, the User namespace was introduced in newer kernel versions,
and with that came a refreshed push for security. Also, Docker came along, and now “containers” were talked about more.
Exciting though Docker was, the team decided against using Docker’s implementation of the Linux technologies because of
how it tightly couples containerisation and UX.

## "Garden-Linux"
And so the Garden-Linux project was created.

At this time the Cloud Foundry team (because it was just one team handling everything) decided to split. New projects were
spawned, each to handle a different component of the platform, and each able to focus exclusively on the specific problems associated they would bring.
Garden-Linux would be the new container runtime, replacing the old Ruby code from warden with Go. ('W'arden -> 'G'arden, geddit?)
The old scheduler, until then called the DEA (droplet execution agent) was also remade with Go into a new component called Diego (DEAGo. We have fun.)

One of the motivations for behind the Garden-Linux project was the wish to have a multi-platform API, which could be run on top of different backends for different systems.
Garden-Linux was one such backend for that API, along with a new Garden-Windows project to support windows based applications on windows servers.

For a while Garden-Linux was implemented fairly similarly to Warden, in that it still used AUFS to provide the root filesystems for containers.
But before long there was a new feature request: customers wanted support for Docker image based applications. And this lead to significant changes
in CF's approach to containers and their root filesystems.

The greatest change lay in CF's control over what sort of stuff gets pushed. When an app is deployed via the standard buildpack model,
CF has control of pretty much everything around it: we own the filesystem, the compilation, the dependencies, and we can even control which user is going to run it.

But allowing the deployment of Docker image apps means CF would now be expected to trust an external directory structure,
which is just about the most untrustworthy thing out there. Furthermore the control over which user runs the application is lost,
as the image spec allows that configuration. This alone forced the Garden-Linux team to up its game and start using user namespaces. A user
namespace allows you to run a container so that it thinks it is running as root, when in fact it is running as something powerless.

So this was great for security reasons, but it meant that we had to abandon our AUFS disk quota implementation. We had relied on enforcing user based
quotas, which was possible when we had control of which user a container ran as. But now that we were using namespaces and mappings, the containers
would now all be run as the same user. A new solution had to be found.  

### BTRFS?
By now Docker was mature enough that Garden-Linux was comfortable wrapping parts of its API, namely the Graph Driver to handle the filesystem bits of the container creation.
But because we were using the graph driver, we could only use filesystems that it would support.
So from the supported filesystems Docker made available, BTRFS seemed to work best for our case. All the others lacked maturity, lacked required features, or
could only be used under license.
The fact that BTRFS had native quota support was a bonus, so we went for it.

However, a new filesystem driver deep in the stack was not the only new thing happening to CF components.
Even if Diego and Garden-Linux had already been tested in production environments, using the old AUFS implementation, they were still quite new.
Along with the switch to BTRFS came a new IAAS; this was the first run on AWS. 
And funnily enough this combination proved a little problematic.

There was a huge performance drawback: the VMs that were running the applications became unresponsive.
After a collective effort to discover the cause, eventually BTRFS was blamed.
The theory was that the BTRFS garbage collector was consuming all the disk’s available IOPs, thus causing the system to starve. 

The team was once again forced to find a new solution.

### AUFS. Again.
The team quickly switched back to a file system they were familiar with and could get working again quickly.

The plan was still not to stay on AUFS, however; there was still no mainline support, and still no native way to control
application disk quotas. The old way of using a different user for each container no longer worked because,
since user namespaces came along, every container was run by the same user.

For now, the team managed to do quotas like this:

![alt text](/assets/images/aufsloops "aufs loopback quota hack")

A formatted sparse file fixed to the requested app quota size was
attached to a loop device and mounted on top of the AUFS mounted root filesystem. A loop device is really a normal
file which when mounted acts like a block device: something with access to actual hardware or disk. An application
could therefore not exceed the size enforced by the truncate, because the loop device believes that the size of its
backing file is the total size of the disk it thinks it is writing to. Although this trick solved the quota problem,
a max of 255 loop devices wasn’t great for scalability.

So that was the state of Garden when, in 2015, runc was released by the Open Container Initiative. Runc provided a
lightweight and daemonless way to run containers and gave greater flexibility for broad use cases. This was great news
for CF because by getting Garden to work with runc we could have the best of both worlds: we get the best bits of Docker
while complying with CF’s user experience and far less maintenance overhead. It also meant that we would get more opportunity
to contribute back into the OCI community.

The Open Container Initiative created the Container Spec, an open standard to describe containers, and also the Image Spec, the
same thing but for images. 
Runc is built to be OCI compliant, and is the building block for most of these container today.

## "Garden-Runc"
So not long after runc was released into the wild, the Garden-Runc implementation started. 
Garden would no longer have to deal with ancient C code, wrapping syscalls and dealing with obscure low level bits.
Instead it would now wrap runc, which should abstract most things related to container creation.
As part of the new runc implementation, CF was also pushing more security features into Garden. The new target at this time was to run
containers completely in [rootless mode](/blog/the-route-to-rootless-containers).
That meant that the Garden binary would no longer have root privileges, which means it would no longer be able to create privileged containers.

But we were still using AUFS. AUFS can not be mounted without root privilege.
That AUFS was a huge distraction for the team was becoming increasingly obvious. Instead of focusing on the new security features,
Garden was wasting days patching up AUFS every time the kernel was bumped or a new bug surfaced. Most of our support calls were somehow related to AUFS.

Fortunately, the Image Spec created by the OCI opened up new possibilities for replacing the old graph driver code.
The Garden team was split in two, and the Garden RootFS team was created (Grootfs) to focus on a new solution for the container root filesystem problem.
This team would focus on the FS side of the security track, by designing a solution which could be run rootless.  

### "GrootFS"
And this is where Tiago and I came in. GrootFS was designed to be a daemonless plugin from which Garden would request a root filesystem for its containers.
The CLI tool can be used independently of Cloud Foundry to create a root filesystem from either tarball containing a customised structure or
a Docker image both of which are then mounted and controlled by a filesystem driver.
Because there was a push to get off AUFS, the filesystem Groot was initially built around was BTRFS.

### BTRFS. Again?!
Yes BTRFS again! You may be wondering why we would go back to this after the disastrous time we faced a year before.
Back then, as I alluded to above, we weren’t entirely certain what was causing those performance issues, but we had a pretty good
working theory that it was caused by BTRFS’ cleanup process consuming all available IOPs.
To ensure it didn’t happen again, we proved that the original problem found on a 3.16 kernel, did not happen on 4.4 and,
after extensive testing, decided that we should go for it.
The factors which had helped BTRFS progress and mature since a year before, were big company investment and extensive testing
in production environments. On top of that it also had support from Canonical, which meant that CF could have more confidence
in BTRFS as a production ready filesystem driver.

Aside from that stability, the features which had lead to us to choose BTRFS the first time were still things CF wanted.
Namely, BTRFS’ built in quota support and its ability to be, for the most part, rootless which could one day be implemented
in true unprivileged containers. The other useful feature was BTRFS’ snapshotting mechanism which works extremely well with image layering.

Here is illustration showing the two different ways your application could now be run in CF using Garden-Runc and GrootFS: 

![alt text](/assets/images/buildvsdocker "buildpack vs docker apps")

When you pull a Docker image it comes in various tarred layers which must be unpacked and reassembled in a specific order.
Snapshotting provides a simple way to build up each layer upon the previous one without needing to duplicate any of the files.
As many images have layers in common, this is very useful.

With BTRFS, Groot unpacks the base layer and snapshots it. Then on that snapshot it unpacks the next layer, snapshots and so on.
Finally the last snapshot ends up being the read-write layer for your container.

On the left is the buildpack route: here there is only one layer because the rootfs used for buildpack containers comes from a single tarred file,
which is then unpacked and snapshotted to produce the read-write layer into which the app code is placed.
On the right is a Docker image based app, in which each layer is untarred and snapshotted in order with the last layer being the application layer.
Note that here there is no application code bundle because, of course, the app is built into the image as the final read-only layer.

But… it didn’t go as smoothly we’d hoped. This time, however, we were ready: good continuous integration meant performance issues were caught quickly.
It turned out that simply enabling BTRFS quotas on a filesystem had a serious negative impact on performance.
Because CF is a multi-tenanted platform and sets resource quotas on every app by default, there was no option to turn off this
functionality: quotas were essential. So we had to find yet another way.

### OverlayFS + XFS
Which takes us to the current implementation of Garden-Runc with GrootFS: Overlay and XFS.

We use OverlayFS in a very similar way to how we used AUFS: by combining the different layers of an image, and placing a thin read-write layer on top of it.
OverlayFS has no quota tooling, which is where XFS come in.

Below you can see how we use the two together:

![alt text](/assets/images/overlayxfs "overlayfs and xfs")

The base of our graph (aka the place where we store all our image layers and keep the container rootfses) is formatted
with XFS. The loop device it is mounted on is truncated to use just less that the `data` dir which is its base.
Then within that XFS filesystem is each container's unique rootfs, mounted by OverlayFS. The rootfs dir itself has a
directory quota applied to it by XFS.

Aside from quotas, the team went with OverlayFS due to a fairly recent decision by Canonical to release a patched version of Ubuntu in which 
it is possible to perform mounts as an unprivileged user in a namespace, thus meeting our new security mandate.

As you can see, there wasn’t a straight line from the beginning to here, and it is very unlikely that our current choice is where we will stay. ShiftFS is
starting to look good to us so watch this space.

## Try this at home!
In the meantime if you want to make some filesystems yourself you can do so. All Garden’s, Groot's and Cloud Foundry’s code is on github, all open source.
- [garden-runc-release](github.com/cloudfoundry/garden-runc-release)
- [grootfs](github.com/cloudfoundry/grootfs)

If you are deploying via CF, GrootFS is already enabled by default!
