---
layout: post
title: "talk idea: all the weird stuff that comes with pulling the kernel out from under you"
---

great things come with new kernels
- seccomp
- apparmor
- user namespace
- better filesystems

but also weird things
- overlay regression 4.15
- kernel memory leak (udev) 4.10
- creation getting slower over time 4.4
- srcu data race
- named cgroups 
- systemd messing with cgroups



Kubecon CFP:
Description:
We all know about the stuff containers are made of. All those nifty little features in the kernel which lead to whole conferences like this existing.
And these features keep getting better, alongside new ones, with each version. More namespaces, more filesystems, more security, oh my!
But it's not all smooth sailing. The kernel is responsible for much more than isolating cute Docker apps, and often new features appear which make life... interesting for runtime providers.
Such is the case for Garden: the team behind the container runtime for the Cloud Foundry Platform as a Service.
In this talk, Claudia will talk through some of the unexpected, and often frustrating, new behaviour her team encountered when jumping across major kernel versions.
From memory leaks, to SRCU data races, to whatever the hell Systemd is up to, she will share how her team investigated and solved these problems.

Benefits to ecosystem:
This talk would be of interest to those who use or manage container runtimes, but would also like to know a bit more about what is going on "under the hood".
Although I suspect the majority of interest to come from operators, all attendees will find something of use. All concepts will be explained
in a way that will hopefully be high-level enough, despite being on a low-level subject, for all to understand.
Hopefully operators and deployers of containers, especially at scale, will benefit from learning the core role the kernel plays in container technology
and will know either to use kernel versions / OS distributions vouched for by the runtime providers, or to be on the lookout for some strange occurrences.
