---
layout: post
title: "The Route to Rootless Containers"
date: "Sept 08, 2018"
---

This post is a loose transcript of a talk I gave at [Container Camp UK 2018](/placeholder), in
which I described my team's particular journey to true rootless containers.

I work on the team which provides the container runtime for an Open Source Platform as a Service
called [Cloud Foundry](https://en.wikipedia.org/wiki/Cloud_Foundry). CF is popular with enterprise and banks etc and a common comparison people
make is with Heroku. The main difference between the two is that CF is open source, so you can see the code
and hack on it yourself and build your very own cluster on a set of GCP/AWS/whatever instances.

Of course, you rarely build a whole platform for just you and your instagram ripoff.
What you are more likely to do, is let other developers push their apps to your cloud, where you run them for a reasonable fee.

But this is a big risk on your part. As soon as your platform goes public, anyone can buy space
and push their code or run their docker images (which could contain just about anything) and you
don’t know who they are and what they are sending to your precious platform.

This is why on the container runtime team for CF (we’re called Garden by the way but
that’s a whole other story) we always plan for the worst case scenario. And we always strive
to be on the bleeding edge of container security.

## Containers?
So what are containers? (feel free to skip ahead if you already know this.)
When new engineers come onto the Garden team at [Pivotal](https://pivotal.io/), the first thing we veterans say is that containers _don’t exist_.
We say this very smugly and pleased with how clever and witty we are, but it is true after all.
Containers are just processes wrapped in abstractions over kernel tools and primitives.

The imagery of a "container" or box-like thing is very misleading. Maybe a colander is more accurate?
Or a hall of mirrors except the mirrors are those two-way ones popular with interrogators and the interrogators
on the "knowing" side of the glass are telling the person on the other side how the world is and that person is
totally believing them? Or the Matrix? Trapped by nothing more than belief in an illusion? (Ignoring the world dominating AI).
Yes let's go with the Matrix.

The things which make these processes sort-of “contained” in the Matrix and different from just any other process,
are isolation and their view of available resources.

By isolation we mean they are able to run without interference _from_ other processes and without
interfering _with_ other processes. By available resources we mean that we ensure that they are only
able to exploit the subset of resources (so memory, cpu) that we expose to them. Lastly we ensure
that they are not able to alter those limitations. And we do this by using things given to us by the kernel:
[namespaces](http://man7.org/linux/man-pages/man7/namespaces.7.html) and [cgroups](http://man7.org/linux/man-pages/man7/cgroups.7.html).

But it’s not just the containers themselves which need to be restrained and limited. What about the thing running those containers,
orchestrating those lifecycles? This means things like the Docker daemon or in CF's case the Garden Server.
Until fairly recently, these components have had to be run with superuser or root privileges. And why is this dangerous?
Because should something escape from our sort-of contained container, it could then have root access to the whole system which is not ideal.

Before I get to how we (Garden) managed to run ourselves rootless, I will go over the couple of things which have to happen first:
- The things which make a linux process "contained" in the first place.
- How those things are used by Garden to make that "contained" process run without privilege.

And then I can get into:
- How Garden can make itself run without privilege (while still making containers without privilege).

The things I am going to talk about in point one are the standard things which make a container a container,
they exist in the kernel and everyone used them. Using those things in a rootless
way is what makes container providers stand out when it comes to security... but few are really even doing that right now.
(Garden is, but Docker still has a `--privileged` flag -.-). Most of the tools needed to do rootless containers are in the kernel, you just need
to do a little work to get to them.

The last bit, making orchestrators run rootless, is hot right now but so far as we know,
Garden is the only team ready to do this in production. However, this is not just about our journey and as I go through ours, I will namecheck
those others which intersect.

## What makes a container a container?
So things which contain a container. We’ll start with namespaces.
### Namespaces
Namespaces allow you to isolate global system resources for a process. These resources are things like mountpoints, network devices, process ids etc.
When you put a process in a namespace, you can make that process think that it has exclusive access to those resources.
There are 7 namespaces in linux right now: Mount(mnt), Interprocess Communication(ipc), Process ID(pid), Network(net), UTS, Control Group(cgroup) and User ID(user),
and this last one, the User namespace, is particularly relevant when it comes to rootless containers so we will come back to this in a moment.
Other namespaces which containers make significant use of are the pid and mount namespaces so let's go into them a little deeper.

#### PID Namespace
With the pid namespace 2 processes running on the same host but in a different PID namespace can share the same PID,
or at least appear to. A process inside the child namespace may think it is PID 1, but really it is mapped to a higher
PID in its parent namespace and has no knowledge of PIDs outside it’s space.

![alt text](/assets/images/pidns "pid namespace")

#### Mount Namespace
Process which have been put in a new mount namespace have a different view of which points are mounted.
When [pivot_root](http://man7.org/linux/man-pages/man2/pivot_root.2.html) is also used, that process will see something different mounted at /. Instead of seeing whatever the rootfs
is in the host or parent namespace, it can be made to see its own rootfs. This appears to the namespaced process to be at /,
when it is probably somewhere else in the parent namespace.

![alt text](/assets/images/mountns "mnt namespace")

In the example above, I used `df` to show the mountpoints because the output is nicer for slides/images. I would usually
look at the mount table via `cat /proc/<pid>/mountinfo`.
One way which you can see the namespace your process is running in is by doing `ls -l /proc/<pid>/ns` and seeing the id.

### CGroups
If namespaces alter a process’ view of the system and its resources, then cgroups enforce what a process can
actually do with resources. Control groups are used to limit the container process’ use of things like memory or cpu.
The other resource we care about which cannot be handled by namespaces or cgroups is disk quota: we don’t
want containers to be able to write whatever they want, flooding the disk and ruining it for everyone else.
This is very important for a multi tenanted system like CF. This we deal with at the filesystem level and it
caused us a significant amount of grief and I am going to get into that more below.

### Dependencies
The last thing which makes a container the thing we are all very familiar with and different from a
standard linux container is the encapsulation of dependencies. A few years ago this little company
came along into the container ecosystem and made the teeniest tiniest change to how we see and work with 
containers. They were called Docker and they figured out a way to encapsulate dependencies in shippable
units within things called images, and they figured out how to do this efficiently. These images could
be moved around between machines which meant we could run many many different containers based on these
identical blueprints which underneath were using the same linux primitives.
And this is the key difference between linux containers and the containers which we talk about today:
Isolation gives you a linux container, but isolation along with encsapsulation gives you a “container”.

So how does this work? How do we package dependencies up into something which is quick and safe and
efficient to ship around? To do this we used a layered filesystem and pivot_root. Let’s start with how
pivot_root works.

#### pivot_root
`pivot_root` is a system call which changes what a process see when it looks at `/`.
In practice it works like this:
Say you have a program called run.sh and this program asks “what’s in /?”. 
The files which come back will be the host rootfs, some boring ubuntu or whatever.
But then the program can run pivot_root with a path to somewhere else,
and the next time it asks “what’s in /?” it will see a busybox or alpine or whatever filesystem.

![alt text](/assets/images/pivotroot "pivot root")

#### Layered Filesystem
How would a container end up with this filesystem? That is where layered filesystems come in.
One way we (and by we I mean the community as a whole) used to deal with this packaging of dependencies
was to wrap up everything into a VMDK or an AMI or whatever which works fine... if you don’t care about
things like boot speed and keeping your disk free of duplicate files. Imagine you had one application
which relied on Ubuntu for it’s base filesystem stuff, and you also had another application which also
asked for an Ubuntu fs… well you’d end up with two whole Ubuntu filesystems, which are not small, on your drive.
And then you ask for another Ubuntu based app, and you see how this is hugely inefficient.
And this is where Docker really saved us a lot of pain. They realised that this thing called layered filesystems,
which had been around for ages of course, could be used to get containers their dependencies.

How does this work? Picture a standard Dockerfile: the first line begins with `FROM: ` something, like Ubuntu or whatever.
When a container’s orchestrator reads that line, it will download and cache the Ubuntu filesystem to a single read-only dir.
When the next application comes along and asks to be run in a container also backed by Ubuntu, 
the orchestrator will think “Oh snap, I already got one of those”.
Any other lines in your Dockerfile, ADDing application files, RUNning bundle apt installs, end up in other read-only directories,
which will then get layered together by some COW filesystem to give a unified view, the top read-write layer of which becomes
your container’s rootfs, visible at `/` thanks to `pivot_root`. So layered filesystems allow us to share common things between containers.

![alt text](/assets/images/layeredfs "layered filesystem")

## What makes a container an unprivileged container?
So now that we have isolated and controlled our process and its resources, how do we stop that process
from de-isolating itself and removing those limitations? The first thing we can do is ensure that if
that process can de-isolate itself, it doesn’t actually have the capability to do anything of significance
in the parent namespace. In the past, you could be one of two things on a linux machine: you were either root,
or you weren’t. Root or superuser essentially meant you were in God mode and had the power to do anything you liked;
nothing was off limits. And if you weren’t root, if you were just a mere mortal, your options were limited and boring.
Nowadays, instead of having all the power concentrated in one role, these abilities have been split into a variety of entities.
And these are called capabilities or CAPS.

### Capabilities (CAPS)
Some examples of [capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html):
- `cap_set_uid` lets you change uid and write userid mappings in a user namespace.
- `cap_net_bind_service` lets you listen on privileged ports.
- `cap_kill` means you can send any signal to any process.
- `cap_chown` lets you chown any file.
- And then there’s `cap_sys_admin` which you can tell by the name let’s you do a lot, it’s a little bit overpowered.
If you look on the man7 docs, every other cap has a list of 1 or 2 (no more than 5) things which it can do, whereas
the list under `cap_sys_admin` is long with more than 2 dozen powers listed.

So when it comes to a container process we can just drop the more powerful caps, so that while it has the ability to
do what it needs to do inside the container, if it were to break out of the isolation we have created it is pretty much powerless.

### Seccomp
The next line of defense we have is [Seccomp](http://man7.org/linux/man-pages/man2/seccomp.2.html), aka secure computing mode. This is a built-in kernel thing which was merged
upstream for version 2.6.12 back in 2005. Seccomp lets you either limit system calls available to processes or restrict
the flags which can be set on those calls. An example of this would be; allowing containers to make the clone syscall,
which they would need to create new processes but then limiting the flags they can use with that syscall so that they
cannot create that process in a new namespace. 

### Apparmor
And then there is [Apparmor](http://manpages.ubuntu.com/manpages/xenial/man7/apparmor.7.html) which is a Mandatory Access Control system. Again this is a way to confine programs to a
limited set of resources. This confinement is dictated by profiles which are loaded at boot time so they cannot be
turned off while the system is running. The profile is a text file in which rules are written to DENY process’
ability to write under certain file paths.

## _Shoutouts Sidebar!_
_Of course this route was only possible due to the work of some very clever people in the community,_
_if we couldn’t call our wrapped components, like runc, as a non-privileged user, then we couldn’t ever_
_hope to call Garden as an unprivileged user. So I need to call out a few in particular to whom we are all very indebted:_
_[Jess Frazelle](https://blog.jessfraz.com/), [Aleksa Sarai](https://www.cyphar.com/), [Akihiro Suda](https://github.com/akihirosuda), and many more._

_And I also want to mention [runc](https://github.com/opencontainers/runc) in particular. In the past the Garden team was often asked, why not use or wrap Docker?_
_Why write our own container runtime? And the answer was a) we started doing containery stuff before Docker existed and b)_
_when Docker did get going, although it was cool, we couldn’t use too much of their API because it was very heavy and extremely_
_opinionated in a way that made it too rigid for Cloud Foundry’s use case._

_Then in 2015, runc was released by the Open Container Initiative._
_It came from something which Docker developed to replace LXC, and I think has been used inside Docker in one way or another since_
_version 1.11. Runc provides a lightweight and daemonless way to run containers and gave greater flexibility for broad use cases._
_This was great news for CF because by getting Garden to work with runc we could have the best of both worlds: we get the best bits_
_of Docker while complying with CF’s user experience and far less maintenance overhead. We were also able to contribute back into the OCI community._

_And we are very grateful to the community and the OCI. We are now saved a lot of heavy lifting by runc, except in the case of_
_images and filesystem layers which we handle ourselves. So we do appreciate that we can rely on the work done by a lot of very talented people._

## All good?
So are we good? With all that, with the isolation, the encapsulation and the restricted resources,
are Garden containers totally secure and bulletproof? Well… sure, so far we have suffered no major
exploits. But that doesn’t mean we are invulnerable. If you picture the various exit points of a container as doors,
yes we have made a pretty solid door. But how likely are attackers to stop there? Pretty unlikely. So they will look elsewhere, 
they will look at the walls and they will find windows.

## What makes a container orchestrator unprivileged?
Going back to the things which make a container a container, I mentioned that the User Namespace would be important.
### User Namespace
The introduction of the user namespace in Linux 3.18 was a very important step for containerisation technology.
It lets us make a container process think it has a full set of init root powers, where it actually has nothing like it.

That’s the container itself though, how does it relate to running the container orchestrator (Garden) unprivileged?
Well the key thing is: you don’t have to be privileged to create a new user namespace. This is great, because it means
that Garden does not need to be running root or privileged in order to create those user namespaces for its containers,
it just needs the right caps. This is safe because once you are root in your namespace, you only have root in your namespace,
not in the init namespace.

So how do user namespaces work? They make the user think they are more powerful than they are. In the container,
the root user thinks it is all powerful and brilliant. It can’t see outside its user namespace, its own little world,
and therefore can’t comprehend that there are bigger, more powerful players out there.

A colleague of mine likes to use Fight Club as a metaphor for user namespaces. In the real world, the user is boring
Ed Norton, but in its own namespace, it gets to be cool Brad Pitt.

I however prefer to use the last scene from Men In Black 1. Earth thinks it is all big an important, but really it is just a tiny
speck in a galaxy which is some alien's toy.

![alt text](/assets/images/marble.gif "user ns metaphor")

#### User mappings
We create this illusion of power by mappings: the user in the container is mapped to a real user in the init namespace, a random one with zero capabilities.

In Garden we always map container root to the maximum ID on the host (4294967294 on 64bit machines)
as this user is the least likely to have any dangerous capabilities in the init namespace, so it is
safe to map this to 0 in the container namespace. We call this user maximus, obviously, and this is
the user we run the garden server as when we run rootless.

By default you only get this one uid mapping, the user you are, which ends up being root in the container namespace.
There are no other users in the container namespace because those mappings have not been set in the host.

This is deliberate: you don’t want a situation were other uids are automatically mapped from the host to the container.
Were this the case then all you would have to do to gain privileges as an unprivileged user, would be to create a new user
namespace and then become a user in that namespace which is mapped to a privileged user in the init namespace.

However in our containers we do actually need more than one user, so we do want to map some. But we want to do it carefully.
We rely on tools within the shadow package called `newuidmap` and `newgidmap` which have a `suid` bit set on them. Setuid means
that whichever user executes these files, will change user to whoever owns those files for just the duration of the execution.
`newuidmap` and `newgidmap` look into `/etc/suid` and find out which uids and gids a user is allowed to map and then writes those mappings for you
in the process' `/proc/<pid>/{uid_map,gid_map}` files.
We PRed the initial version of support for this back to runc.
The extra mappings we choose are, like maximus, least likely to be anything actually important in the init namespace.

Here is an example of what is is written to a container’s uid_map file. Our id map files are written with 3 values on each line.

![alt text](/assets/images/mappings "user ns mappings")

The first line here you can see is the root mapping. The first value is the user’s id in the container namespace, 0,
the second value is the user’s id in the parent namespace, maximus, and the third value is the range of ids we want
to map or the number of ids we want to map starting from that point, in this case just one; root.

On the second line we map the subordinate ranges, the other users which we could become in the user namespace. 
Of course, we don’t want these to translate to anything important in the parent or init namespace, so we are starting
from somewhere higher than nobody, as the uids higher than that are probably not up to much. Again, the first number
is the uid inside the container namespace we want to start mapping from, the second number is what that user will be
mapped to in the parent namespace, and the third number is the number of uids we want to map from that starting point (so maximus - 65536).

### CGroups (rootless ones)
The next element we wanted to get rootless was our control groups. There is no clean way to do this yet because
cgroups are a virtual filesystem, with all files owned by init root so the container root, host maximus, does
not have power to do anything. We managed to get around this by putting some cgroup setup into a program which
would run as a privileged user before we start the server which handles container lifetimes as an unprivileged user.
This setup simply creates a subdir in the cgroup hierarchy, and then chowns those directories to be owned by that
unprivileged user; maximus. We PRed runc so that it could make use of cgroups set up in this way.

### RootFS Mounts
I talked above about how encapsulation is the thing which makes containers nice and portable and stops
their dependencies from flooding you disk, and this is done through filesystem layering. Well, layered
mounting often requires root, depending on the filesystem you choose, so that was our next target.
Just to quickly clarify what we mean by a filesystem: there are 2 meanings associated with this word.
One is the directory hierarchy used to store data, the other is the type of filesystem which determines how that data is organised.
Here I am going to be referring to how we use types of filesystems to create the filesystem structure used by containers.

#### AUFS
Back in the day, when Garden was young and not even called Garden, we used AUFS to create those layered
filesystems for containers. AUFS is a filesystem type which implements a union mount to combine several
directories to make their contents appear to be under just one directory. And we used AUFS for ages and
it was a headache, the only reason we didn’t choose something else, was because the other options were
either more of a headache or were under proprietary license. Our main gripes with AUFS were a) it was
not in the mainline kernel and including that module would go wrong more often than it went right, b)
it is maintained by exactly one person and c) you have to be root to do an AUFS mount.

#### BTRFS
But a couple of years ago another option became very attractive, so we decided to create a whole new
team dedicated to giving us rootlessly created root filesystems. We called it GrootFS, which was a
terrible name given that we did not actually make a filesystem just a filesystem maker, and we chose
BTRFS for our new filesystem layerer. BTRFS looked good to us because a) it has disk quota tooling built in,
is very important for a multi-tenanted system and b) aside from needing some privilege for setup,
it could COW snapshot as an unprivileged user. We were very excited about this.
It turned out to be too good to be true however. Among other things, BTRFS’ disk quotas do not do well under load and the whole platform ground to a nasty halt.

#### OverlayFS
Within a couple of weeks we turned it around with our new favourite filesystem, OverlayFS. OverlayFS
is similar to AUFS in that it is a Union type filesystem. It is possible to perform overlay mounts as
an unprivileged user… in certain environments. Canonical Ubuntu had a chat with the OverlayFS maintainers
and decided that _could_ be acceptable, and not a security risk, for an unprivileged user to do overlay mounts.
Ubuntu therefore compiles a patch kernel with Overlay mounting in unprivileged userns enabled. This means that our containers can have their
rootfs mounted with overlay in their unprivileged namespace… only on Ubuntu. Which does tie us down a little.

So how do we make this work? Well, we still use GrootFS to pull down and unpack all the layers of the filesystem
and put them into a directory which is chowned to the maximus user. GrootFS is running as an unprivileged user
outside the container’s user namespace, however, so it will then return all the information that runc will need
to perform that overlay mount inside the user namespace. This information is in accordance with the OCI runtime spec.
Runc will perform those mounts in order, so all we have to do is ensure that the rootfs is listed first, and runc
will take care of mounting that rootfs using the caps which it will have inside that namespace.

![alt text](/assets/images/rootfs "rootfs entry in config.json")

For more on the topic of container root filesystems, please check out [my other post/talk](/blog/container-rootfilesystems-in-prod).

## Most secure container provider out of the box...
So you can see that we don’t rely on just one line of defense; we use everything which is available to us.
Our mission on Garden is to be the most secure container provider out of the box…. Now I am not saying that
we have accomplished this mission yet, but it is what we very much want to be able to confidently say one day.
And it’s not looking too bad right now. If you look at this chart, you can see that we stack up pretty well against the other big players. 

![alt text](/assets/images/secchart "security comparison chart")
<sup>_Last updated 18/02/19._</sup>

## Roadblocks
And that is where we are at now. But we are still not 100% rootless. There are a couple of things still blocking us.

### Disk Quotas
The first is disk quotas. OverlayFS has no built-in quota tooling and this is something we need desperately
for a multi-tenanted public cloud, so we store our filesystem layer cache and container rootfses on a
mountpoint which is formatted with XFS. XFS is another filesystem type which lets us set a disk quota on a
directory… like a container root filesystem. Unfortunately this quota tooling has to be run with root privilege,
so alongside GrootFS we have a little setuid binary, called `TARDIS`, to handle this interaction.

### Networks
Another thing we have no way to do rootless right now is networking. We can create the network namespace
unprivileged and we can create devices in that namespace, but if you actually want to connect to anywhere an
app would find useful, like the internet, then you need to configure the networks on the host and that means you need privilege.
There is progress being made by the community in this area, but for now CF got around this by creating another
separate team and extracting the networking bits into a separate binary which we then, you guessed it, setuid.

### CGroups (even more rootless)
We have already looked at how we do cgroup stuff rootlessly, and that is by setting up when root and
chowning a subdir to our unprivileged user. This is something we hopefully won’t have to do forever
as there is great work being done in the community to solve this. Aleksa Sarai submitted some kernel
patches but the maintainers aren’t entirely sold on making cgroups nsaware, but I doubt he, and others,
will give up. For now Garden can get by with chowning a sub-hierarchy since all the rootful activity
happens before the Garden server starts and long before any containers are created so there is no chance of user input or interference.

## In production?
So Garden has had it’s rootless support ready for a few months now, but because there are many components to Cloud Foundry
there are more teams than just us which need to align. I had really hoped that by the time I did this talk,
we would have been running rootless containers in production, but unfortunately it’s not happened just yet.
We are so close, I can see it on deployment team’s list, so it is painful to think how close I was to being able to say that.
It will be out soon, and hopefully I will be able to update this post with stories of how it all went wrong (I suppose it 
could go right, but...)

## Try this at home!
In the meantime if you want to try out rootless containers yourself you can do so. All Garden’s and all of Cloud Foundry’s code is on github, all open source.
- [garden-runc-release](github.com/cloudfoundry/garden-runc-release)
- [guardian](github.com/cloudfoundry/guardian)
- [grootfs](github.com/cloudfoundry/grootfs)
If you are deploying via CF, just set `garden.experimental_rootless_mode` in your manifest and that’s it: all applications you push will be rootless.

We also have a standalone binary called `gdn` which is available [here](github.com/cloudfoundry/garden-runc-release/releases), and which you can run as an unprivileged
user to get rootless containers, only on that patched Ubuntu of course.

And finally, please check out the [living doc](https://rootlesscontaine.rs/) maintained by Aleksa Sarai on the progress of routless containers.
