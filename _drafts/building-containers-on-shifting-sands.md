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
