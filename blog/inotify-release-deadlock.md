---
layout: post
permalink: /blog/inotify-release-deadlock-envoy
title: "Envoy Proxy Deadlocked My Cloud"
date: "Oct 14th, 2019"
---


I have really procrastinated with writing this blog. A while ago my team at the time, [Garden](https://github.com/cloudfoundry/garden-runc-release),
decided that we were going to start writing more blogs about the stuff we work on and the problems we solve/are mercilessly defeated by. That resolve
started well. I wrote a thing about [OverlayFS](https://www.cloudfoundry.org/blog/an-overlayfs-journey-with-the-garden-team/), my colleague Georgi
wrote one about [SystemD](https://github.com/georgethebeatle/systemdont), and we got lovely feedback from folks on how what we do is interesting and cool.
(The blogs never seem to convey the severe drops in morale which come from _months_ of zero progress, but that is probably for the best, I guess.)

But then we kind of dropped the ball, and nobody has written anything for a very long time (Feb 2019, judging by Georgi's last commit).
Well it was mostly me, to be honest. We had decided that each blog should be written by the person who had spent more time investigating/suffering
than the others on the team. And since I seem to have developed an almost unnatural and sadomasochistic affinity with unsolvable problems, I was often
the one left with the most context. (I also get VERY emotionally invested in investigations and my discipline to let. It. Go. and let others play/struggle
is barely a year old.)

Anyway, it is worse than I thought, since the next post I am meant to write is on a problem I solved _a year ago_.

So without further ado, here is the tale of a time where production cloud instances were brought to a grinding halt by Envoy... or so it seemed!

&nbsp;

----------------------------------

&nbsp;

## _Clouds, Containers, Proxies_

In case this is our first time meeting, the cloud I am talking about is Cloud Foundry (here is a [nice enterprisey link](https://www.cloudfoundry.org/) for you).
It is an open-source platform as a service; if you were so inclined you could use the OSS platform code to deploy your very own cloud (!) on the IAAS of
your choosing, and then host your applications on it / let your friends deploy their apps / charge a fee so randoms can deploy their apps, etc. etc.

The Garden team provides the container runtime (essentially an elaborate wrapper around [runc](https://github.com/opencontainers/runc)) which ensures those apps run in isolation.
We also provide the bonus add-on of "sidecar containers" which lets you associate additional service processes with your application container, without impacting
the main app container's system limits. One of those associated services you can have attached by default to every application container deployed on your cloud is an
[Envoy Proxy](https://www.envoyproxy.io/).

Our sidecar implementation and [Diego's](https://github.com/cloudfoundry/diego-release) (the scheduler) use of them for Envoy weren't brand new things in October
2018, so we were a bit surprised when the lovely people in charge of operating our [publicly available cloud](https://run.pivotal.io/) came to us saying that a pretty
standard deploy had failed on our component's shutdown of multiple Envoy proxies.

Okay so we weren't _that_ surprised. If a problem is going to happen, it usually won't come out of nowhere, but will often appear during a deploy.
Even if our component isn't the one being upgraded, it becomes very 'active' at those times and the surface area for 'weirdness' (from the kernel with which we directly
interact, to the anonymous app code we isolate) becomes more exposed. On most CF deployments, Garden is configured to destroy
all containers when its job/service is restarted. This has no effect on users: their apps are immediately reshceduled to a machine which _isn't_ currently
rolling. But if there are A LOT of application instances on that VMs... then odd things can happen, and we end up with several Github issues which
begin "Deploy failed with...".

On this particular occasion this sentence ended with words which sent shivers down our spines, made our blood run cold, struck fear into our hearts: "...many Envoy processes **_in D-State_**".

&nbsp;

85% of you are asking "_What is D-State?_". The other 15% are thinking "_So what?_".

In normal circumstances D-State is nothing to worry about, a complete non-issue, just a blip that some processes go through now and then.

But this is the _permanent_ kind of D-State, and we had seen its ilk before.

&nbsp;

----------------------------------

&nbsp;

## _States (of D Nation)_

If you have ever run `ps aux` to look for a process you may have noticed that in the (_counts quickly_) 8th column there are letters associated
with each process. (Mostly they are single capital letters, but there are often lots of combinations of lower and uppers and arrows etc., don't
worry about those right now.)

This denotes the process' current state, and during its lifetime a process will be moved between several of these. There are upwards of 12 states
represented by letters in `ps` output and in each processes' `/proc/PID/status` file (depending on your kernel version), but the 5 most commonly seen ones are:

- R: Running or Runnable, it is just waiting for the CPU to get around to it
- S: Interruptible sleep, waiting for an event or a signal
- Z: Zombie, terminated processes waiting to have their statuses collected by a parent
- T: Terminated, processes which have been suspended/stopped
- I: Idle, just chillin, don't need no signal, don't need no cycle time

and finally

- D: Uninterruptible sleep, processes waiting (usually on I/O) to continue, they do not process any signals so **cannot be killed***

<sup>\* _We have found that this is not \*strictly\* true. In some cases we have encountered semi-permanent D-States on which we
have been able to call `kill -9`. I have 3 unsubstantiated hypotheses on this, holla at me if you have the real answer:_</sup>
- <sup>_Those D-States were not "true"; sometimes a state appears as D in `ps`, but they are actually in Z (dead processes awaiting reaping) when `/proc/PID/status` is checked.
Unfortunately I don't recall comparing the two at the time so I am not sure._</sup>
- <sup>_Those D-States were induced by memory related problems, for example; one case was due to a kernel bug which allowed a [cgroup](http://man7.org/linux/man-pages/man7/cgroups.7.html) to use a little more
memory than what is specified in `memory.limit_in_bytes`. Perhaps the kernel handles that sleep differently?_</sup>
- <sup>_The processes weren't in `TASK_UNINTERRUPTIBLE` at all, but perhaps in some other state which the [kernel assigns](https://github.com/torvalds/linux/blob/a2953204b576ea3ba4afd07b917811d50fc49778/include/linux/sched.h#L76-L108)
but which does not have its own representative letter. `TASK_KILLABLE` would be a good candidate for suspicion as it behaves in a similar way to `TASK_UNINTERRUPTIBLE`
while also responding to kill signals. It may have its own letter which I have just not come across yet so_ ¯\\\_(ツ)\_/¯.</sup>

For the most part, your processes will hang about in either S, I or R. If you put a tight `watch` in front of your `ps` command, you may see
processes flickering in and out of the other states, including D. Processes going into D (`TASK_UNINTERRUPTIBLE` to the kernel) is completely normal
and often very deliberate: putting a task into S (`TASK_INTERRUPTIBLE`) means the kernel would then have to watch to see if it has received any signals,
which would be a mess to code and would probably result in some mistakes. (This is fine for something like your shell, which does need to react to just about
anything.) It is far easier to make a process deaf to everything, even a kill signal, except for the one thing it needs to have been accomplished before it continues.

But of course if that _thing_ never happens, then your uninterruptible process can never continue, and with no other way to rescue it the only option
is to reboot the entire machine.

While not the end of the world for this operations team (there are 300+ app-running machines in that cloud, customers rarely notice a thing), it is still
bloody annoying and stressful for them.

D-States are also notoriously difficult to solve, mainly because they are not caused by the thing currently in D: that process is only reacting (or in
the case of being uninterruptible, _not_ reacting).
They are like black holes, is what I am getting at. You can't see it directly, you can only guess what it is up to or that it is even there by how it
impacts other things.

We have been sucked into black holes like these before, which is why, before digging in, we all shared some side-eye and called our loved ones.

&nbsp;

----------------------------------

&nbsp;

## _Nemesis_

Our first, and worst, case of D-State struck in the spring of '17, haunted us for over a year, only to vanish without trace or reason.
The loads on those VMs would increase to 400+ until the machine stopped responding. It was particularly
aggressive, would occur on dozens of VMs at the same time, and contaminated the XFS filesystem where it first appeared: all subsequent process which
went near it would also fall into D. This made debugging very troublesome, since almost anything we ran (`ps`, `lsof` and the like) would get stuck.
Worst of all; any efforts to reproduce outside of production failed.

The operations team eventually wrote a "detect and destroy" script to get the platform back to full capacity as quickly as possible, while we wrote various
data collectors which could gather intel as the problem occurred.  Occasionally
there would be quiet periods, and we would go for a couple of months without a whisper from production, wondering if we had gotten away with it.
Then suddenly a dozen VMs would be brought down in one night. We had theories on everything from complications brought on by [loop devices](http://man7.org/linux/man-pages/man4/loop.4.html),
to a full [XFS](https://wiki.archlinux.org/index.php/XFS) transaction log, to [semaphore](https://en.wikipedia.org/wiki/Semaphore_(programming)) locking bugs.

And then, it vanished. Of course it took us months to relax and close that ticket off. We are still not 100% sure what was going on, but a colleague
is fairly convinced that disabling [transparent hugepages](https://alexandrnikitin.github.io/blog/transparent-hugepages-measuring-the-performance-impact/)
saved the day.

But that was just our first. Before long I was tasked to write a doc entitled "The D-State Glossary", as we had encountered so many kinds with so many
symptoms and had received so many messages from our consumers, that we got very tired of people coming to us with "We've got D-State"... (_"What kind, dammit!!??"_)

Very, very few of these have been solved, but this, dear reader, is a story with a happy ending.

&nbsp;

----------------------------------

&nbsp;

## _Hello, old friend_

The only way to recover those VMs was to reboot, so we wanted to get into the environment asap.
There was no mad urgency but we of course intended to be as fast as possible so that the operations team didn't need to keep it quarantined for long.


To be perfectly honest, despite being constantly crushed under the irresovability of these things, I live for this shit (stockholm syndrome maybe?)
so I was SO EXCITED that it just happend to fall into my lap. I often wistfully think what would happen if I went back to that first D-State with the
knowledge I have gained from all that followed. Ah, we were so young.

Back to the 'present'.

We ssh'ed in to the afflicted VM and ran a variation of `ps` which had proven capable of evading the clutches of the most contagious of D-States:

```sh
~$ ps -eLo pid,tid,ppid,user:11,comm,state,wchan | grep 'D '
   PID     TID    PPID  USER        COMMAND         S WCHAN
609490  609490   609405 4294967294  envoy           D flush_work 
609491  609503   608270 4294967294  envoy           D flush_work
609521  609521   609490 4294967294  envoy           D flush_work
609521  609524   609490 4294967294  envoy           D flush_work
609521  609525   609490 4294967294  envoy           D flush_work
609521  609528   609490 4294967294  envoy           D flush_work
609521  609529   609490 4294967294  envoy           D flush_work
609521  609530   609490 4294967294  envoy           D flush_work
609521  609531   609490 4294967294  envoy           D flush_work
609521  609532   609490 4294967294  envoy           D flush_work
3013281 3013281 3013154 4294967294  envoy           D flush_work
3013281 3013283 3013154 4294967294  envoy           D flush_work
3013281 3013284 3013154 4294967294  envoy           D flush_work
3013281 3013285 3013154 4294967294  envoy           D flush_work
3013281 3013287 3013154 4294967294  envoy           D flush_work
3013281 3013288 3013154 4294967294  envoy           D flush_work
# and so on forever               this column here ^
```

Well that sure is D-State.

We started to go through our standard info gathering steps. First the load averages:

```sh
~$ uptime
11:38:33 up  8 days 22:27,  2 users,  load average: 280.12, 280.13, 280.10
```

Not the highest, but not healthy.

Curiously, according to `top`, those stuck Envoys didn't seem to be using any CPU themselves:

```sh
~ $ top -cbp 3013281
...
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
3013281 4294967+  10 -10       0      0      0 D   0.0  0.0   0:00.05 [envoy]
```

Next we verified that the processes were genuinely in D:

```sh
~$ cat /proc/3013281/status
Name:   envoy
State:  D (disk sleep)
Tgid:   3013281
Ngid:   0
Pid:    3013281
PPid:   3013154
TracerPid:      0
Uid:    4294967294      4294967294      4294967294      4294967294
Gid:    4294967294      4294967294      4294967294      4294967294
...
```

I have mentioned before that sometimes Zombies masquerade as uninterruptible tasks, so I wanted to have a quick check. Obviously I wasn't about to
read nearly 300 processes' status files manually, so:

```sh
# get a count of all Envoy processes in D, according to ps
~$ ps -eLo pid,state,comm | awk '/envoy/ && /D /' | wc -l
291

# filter the pids from that ps output, and use them to read each
# one's stack file. then only count the ones actually in D
~$ ps -eLo pid,state,comm | awk '/envoy/ && /D / {print $1}' | xargs -I {} cat /proc/{}/status | grep 'disk sleep' | wc -l
60
```

So a whole lot of Zombies in disguise, but if we look back at the process output I pasted at the beginning, we can probably assume that those
Zombies are threads of the main processes _genuinely_ in D. A quick check on a thread of the process I looked at earler confirms this:

```sh
~$ cat /proc/3013283/status
Name:   envoy
State:  Z (zombie)
Tgid:   3013281
Ngid:   0
Pid:    3013283
...
```

After separating the actually stuck from the processes which technically didn't exist any more, we turned to the last column of the initial `ps`
output. `WCHAN` is short for 'waiting channel' which is doing exactly what it sounds like: it is something in the kernel that the process is waiting for.

In this case they were all waiting for `flush_work` which is the sort of generic thing I had seen so many times in `ps` output, but it has never been
the source of anything bad, so I had never looked into it before now. A [quick google](https://lwn.net/Articles/286436/) told me that it is a helper which simply waits for the current work
being performed by that thread to complete and be flushed from the workqueue. Cool. So what is everything waiting for?

Our next stop was in the call stacks of one of those Envoys:

```sh
~$ cat /proc/3977870/stack
[<0>] flush_work+0x129/0x1e0
[<0>] flush_delayed_work+0x3f/0x50
[<0>] fsnotify_wait_marks_destroyed+0x15/0x20
[<0>] fsnotify_destroy_group+0x48/0xd0
[<0>] inotify_release+0x1e/0x50
[<0>] __fput+0xea/0x220
[<0>] ____fput+0xe/0x10
[<0>] task_work_run+0x8a/0xb0
[<0>] do_exit+0x2de/0xb50
[<0>] do_group_exit+0x43/0xb0
[<0>] get_signal+0x296/0x5c0
[<0>] do_signal+0x37/0x740
[<0>] exit_to_usermode_loop+0x80/0xd0
[<0>] do_syscall_64+0xf4/0x130
[<0>] entry_SYSCALL_64_after_hwframe+0x3d/0xa2
[<0>] 0xffffffffffffffff
```

Hmmmm. I threw some more `awk` and `xargs` at the 60 confirmed D-States to verify they all had the same stack, and found that they did.

So, they were all calling `inotify_release`, destroying a `group` of some sort, waiting for `marks` to be destroyed, having that work `delayed`,
and here we are.

I had only come across the [`inotify`](http://man7.org/linux/man-pages/man7/inotify.7.html) filewatcher occasionally in vague conversations, so we
planned to settle in to a bout of googling. (It does pretty much what it sounds like: you tell it to watch a file or directory, and it tells you
when certain things happen to that file or directory.)

Just before we got into that, however, we noticed that there was one other process in D-State which was _not_ an Envoy process:

```sh
...
129652 3129652       2 root        kworker/u4:3    D synchronize_srcu.part.13
...

~$ cat /proc/129652/status
[<0>] __synchronize_srcu.part.13+0x85/0xb0
[<0>] synchronize_srcu+0xd3/0xe0
[<0>] fsnotify_mark_destroy_workfn+0x7c/0xe0
[<0>] process_one_work+0x14d/0x410
[<0>] worker_thread+0x22b/0x460
[<0>] kthread+0x105/0x140
[<0>] ret_from_fork+0x35/0x40
[<0>] 0xffffffffffffffff
```

`fsnotify_mark_destroy_workfn` eh? We had found the thing doing the work, so that explained all the Envoys hanging around. But why was _this_ in trouble?

We wondered if all such inotify actions would block on the same worker. We guessed that the simplest command to test this on, without having to read the inotify
`man` page, is `tail -f <filename>`. This command prints live updates in the given file to the terminal, so it made sense that internally it would
be using a mechanism which could watch for various file events. This guess was correct, and we got the following:

```sh
# tail a random file, background it in case the shell gets sucked into D
~$ tail -f foo &

# the process is happily sleeping
~$  ps -eLo pid,state,comm | grep tail
1956364 S tail

# an inotify watch has been added, and it is waiting for events to read
~$ cat /proc/1956364/stack
[<0>] wait_woken+0x43/0x80
[<0>] inotify_read+0x275/0x3e0
[<0>] __vfs_read+0x1b/0x40
[<0>] vfs_read+0x93/0x130
[<0>] SyS_read+0x55/0xc0
[<0>] do_syscall_64+0x73/0x130
[<0>] entry_SYSCALL_64_after_hwframe+0x3d/0xa2
[<0>] 0xffffffffffffffff

# call SIGINT (ctrl-c) on the tail process, background that also, just in case
~$ kill -SIGINT 1956364 &

# the process has gone into D
~$ ps -eLo pid,state,comm | grep tail
1956364 D tail

# and we see in its stack that it is waiting for the same work to be cleared
~# cat /proc/1956364/stack
[<0>] flush_work+0x129/0x1e0
[<0>] flush_delayed_work+0x3f/0x50
[<0>] fsnotify_wait_marks_destroyed+0x15/0x20
[<0>] fsnotify_destroy_group+0x48/0xd0
[<0>] inotify_release+0x1e/0x50
...
```

**Time for theories!**

In order of "least likely" to "most probable":

- Garden-runc is managing mass-teardown of containers badly
- Envoy (which is clearly using the filewatcher in some way) is handling its own teardown badly
- There is a bug in `inotify`, which we have hit by calling `inotify_release` en-masse
- There is a bug in whatever this `srcu` thing is, which we have hit by calling `inotify_release` en-masse

With these theories in mind, we researched both the symptoms (to see if anyone else had hit it), and looked into the three elements which
are not our domain: Envoy, Inotify, SRCU. Our aim was to gather more datapoints which could further our investigation on the machine.

&nbsp;

----------------------------------

&nbsp;

## _In which the internet is stunningly useful_

I was really blown away. Usually when we google our obscure problems' symptoms, we get absolutely fuck all. This time, we actually got some hits of the same thing, maybe.
There was [word of a fix](https://bugs.launchpad.net/ubuntu/+source/linux-azure/+bug/1802021) already out there, so that was nice, but it had not been proven or merged.

Nobody else mentioned Envoy so we were able to tentatively rule that out.
This we confirmed by also reading through [their code](https://github.com/jparise/envoy/blob/553e9aeccf69983d4ac41f3dec72a8bc7b312ca9/source/common/filesystem/inotify/watcher_impl.cc)
calling the inotify interface: they are simply adding watches on files (not sure where they are removing those watches, but perhaps that is not necessary
if the parent process is killed?).

SRCU is a mechanism separate from inotify, so we decided to focus there until we needed to fall back.
It stands for 'Sleepable Read-Update-Copy', and is an extension of the original RCU lockless 'locking' mechanism which allows readers continued
and unbroken access to a data structure, even while that structure is being updated. (Say you were traversing a linked list
and at the same time an rcu updater came in to remove an element in the middle. Your reader, regardless of which part of the structure it had
a reference to, would continue along happily, completely none-the-wiser.)
This mostly exists to cut out the heavy time and physical resource needs of standard locking when synchronising state, and is very cool.
I suggest [having a read](https://www.kernel.org/doc/Documentation/RCU/rcu.txt) or looking at a [diagram](https://en.wikipedia.org/wiki/Read-copy-update)
if you are interested. [_Sleepable_ RCU](https://lwn.net/Articles/202847/) is the same but adds to the 'Classic' by allowing arbitrary sleeping in reads.

The `kworker`'s last known location in the kernel was [`__synchronise_srcu`](https://github.com/torvalds/linux/blob/v4.15/kernel/rcu/srcutree.c#L881-L909), so we
prepared to go digging in that direction in case our thing ended up being unrelated to the issue reported elsewhere. It did appear super similar though,
so our odds looked good.

&nbsp;

First we need to focus on getting a reproduction outside of prod. My hopes for this were fairly low since, as I mentioned earlier, getting non-prod
repros of historical D-States has been something of a White Whale for us. I know that this time we had a solid lead that this thing _was_ reproducible,
but... pessimists are never disappointed.

I decided to create 2 different labs: one running a full Cloud Foundry cluster, and one just a simple Ubuntu on
Vagrant. If it was the same thing as reported and possibly fixed by others online, then the Vagrant should be all I would need in the end. If it was something else,
then I would need to re-add all the variables and gather some wider information, so it would be good to have that full env ready to go, just in case.

On the full CF I kicked off a fairly naive, and fairly blunt, loop which would create and destroy hundreds of applications. Nothing fancy there. When that was fine to be left alone
I set up a simple VM running Ubuntu Xenial, [bumped the kernel](/assets/gists/switch-kernel) to match the one running in production (4.15), [installed Envoy](https://www.getenvoy.io/platforms/envoy/ubuntu/),
fetched some [sample config](https://www.getenvoy.io/tutorials/envoy/getting-started/front-proxy/), and started a proxy to verify that inotify was in use by Envoy.

```sh
~$ lsof | awk '/inotify/ && /envoy/'
# tumbleweed
```

Huh.

I tried a different tack. I attached an `strace` to the running Envoy process, killed it, and grepped all child process files for mentions of inotify:

```sh
~$ strace -y -yy -v -t -ff -o str-envoy -p 56240 &
~$ killall envoy
~$ grep inotify str-envoy.*
# more tumbleweed
```

So it seems that bog-standard Envoy does not use inotify, but there is clearly some feature we are using in production which does, otherwise those processes wouldn't
end up in the state they are in. Going back over my research notes, I recalled that it is when using a Discovery Service of some kind that their inotify wrapper
will be used to watch for changes in configuration for that service. I took a quick peek at an `envoy.yaml` config file being used by a running Envoy proxy in my full CF test env,
and saw that it did indeed have an SDS ([Secret Discovery Service](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret)) set up, with keys pointing
to configuration files.

I grabbed that Envoy config file, as well as the two SDS config files, and this time when I started the proxy I could see inotify associated with
the Envoy process listed in `lsof`:

```sh
~$ lsof | awk '/inotify/ && /envoy/'
envoy     3997            root   27r  a_inode               0,13        0      10465 inotify
envoy     3997            root   28r  a_inode               0,13        0      10465 inotify
envoy     3997 4001       root   27r  a_inode               0,13        0      10465 inotify
... # 11 more

~$ strace -y -yy -v -t -ff -o str-envoy -p 3997 &
~$ killall envoy
~$ grep inotify str-envoy.*
str-out.3997:11:56:52 close(28<anon_inode:inotify>)  = 0
str-out.3997:11:56:52 close(27<anon_inode:inotify>)  = 0
```

That's more like it.

We stripped down all the config files to the bare minimum needed to get the proxy running with inotify also in use, and kicked off a loop which
started 100 proxies, then killed them all; essentially the same thing as running in my heavier env, but with a _lot_ less overhead.

Why didn't we just use a loop around inotify? Wouldn't it be faster? Isn't _that_ the closest thing to the root cause anyway? Correct, but I wanted to see
a reproduction as close as possible to the thing seen in prod, so I didn't want to tear out everything.

Nevertheless, we did decide to also write a [little program in Rust](https://github.com/Callisto13/srcu-race-repro/blob/master/rust-test/src/main.rs)
which would add and then remove hundreds of inotify watches on a file. We set multiple while loops off to run this in another light env. (Yes, another.
I like to be sure, assumptions are the root of all evil, and local Vagrants are cheap, so.)

After a couple of hours our environments hadn't triggered anything, so we hit up [Canonical](https://canonical.com/), from whom we get the base images for our platform operating system,
to see if they were aware of the problem, if the rumoured fix was good, and just to get their general thoughts on our issue.
We are also _not_ kernel contributers, just enthusiasts, so delving in and patching the problem ourselves wasn't something we could take time
out of our _actual_ jobs to do. (And to be honest, RCU code is _**intense af**_ so hell knows how long that would have taken us!)

&nbsp;

Canonical knew immediately what I was talking about (they are pretty good at saving us), and sent me a [link to the thread](https://lists.gt.net/linux/kernel/3124443)
on the Linux Kernel Mailing List in which Dennis Krein had discovered, and then proposed the patch, to the issue we had read about others experiencing.
The asked me to verify, and saved me masses of time by backporting the patch to the kernel I was using and pointing me to the PPA.

Awesome! Now all I had to do was reproduce the damn thing. Before reaching out to Canonical, I had assumed that the reason various loops hadn't succeeded
was because they were not doing the right thing, or we had been wrong all along. Turns out I had just been impatient. According to Krein's first report:

>I have done this repro scenario 4 times and have hit the bug within 12 hours or less each time - once in only 2 hours.

Fine then, maybe "a couple of hours" had been a bit optimistic.

Krein's had used a modified version of Michael Kerrisk's [`demo-inotify.c`](https://github.com/bradfa/tlpi-dist/blob/master/inotify/demo_inotify.c)
program from the Linux Programming Interface in his reproduction. Just in case our Rust _was_ incorrect, I grabbed the edited version, [modified it further](https://github.com/Callisto13/srcu-race-repro/blob/master/inotify_test.c),
and threw that into the fray in the third environment as well.

His repro also involved something called [`rcutorture`](https://landley.net/kdocs/Documentation/RCU/torture.txt) which sounded cool, but
didn't seem to have been enabled in my kernel. I decided not to enable it for now, as I felt I had enough going on. I had even scaled back a bit after running out of memory
and hitting [`fork()`](http://man7.org/linux/man-pages/man2/fork.2.html) errors (aka new processes could not be created). Oops.

&nbsp;

----------------------------------

&nbsp;

## _The long game_

Since I was settling in for a lot of waiting, it seemed a good time to read what exactly was up with SRCU.

The commit message with [Krein's patch](https://git.kernel.org/pub/scm/linux/kernel/git/paulmck/linux-rcu.git/commit/?h=dev&id=1a05c0cd2fee234a10362cc8f66057557cbb291f) reads:

>The srcu_gp_start() function is called with the srcu_struct structure's
>->lock held, but not with the srcu_data structure's ->lock.  This is
>problematic because this function accesses and updates the srcu_data
>structure's ->srcu_cblist, which is protected by that lock.  Failing to
>hold this lock can result in corruption of the SRCU callback lists,
>which in turn can result in arbitrarily bad results.

So a data race. The offending block of code can be found [here](https://github.com/torvalds/linux/blob/v4.15/kernel/rcu/srcutree.c#L410-L425).
As Krein explains, the lock held on [this struct](https://github.com/torvalds/linux/blob/v4.15/kernel/rcu/srcutree.c#L415) is all well and good,
but since [this structure's](https://github.com/torvalds/linux/blob/v4.15/kernel/rcu/srcutree.c#L412) callback list 
(aka a list of functions queued up to be called at a later moment) [is being updated here](https://github.com/torvalds/linux/blob/v4.15/kernel/rcu/srcutree.c#L417),
there is no guard against another thread altering it out of turn. The current head of Linux [contains the patch](https://github.com/torvalds/linux/blob/7e67a859997aad47727aff9c5a32e160da079ce3/kernel/rcu/srcutree.c#L438-L455)
with that struct locked down.

<sup>_(`srcu_gp_start` is called from within [`srcu_funnel_gp_start`](https://github.com/torvalds/linux/blob/v4.15/kernel/rcu/srcutree.c#L614-L671)
which is called from [`__call_srcu`](https://github.com/torvalds/linux/blob/v4.15/kernel/rcu/srcutree.c#L814-L852), the call to which comes
from the place we last left off in the kernel: [`__synchronize_srcu`](https://github.com/torvalds/linux/blob/v4.15/kernel/rcu/srcutree.c#L881-L909))_</sup>

So now we know how parallel `inotify_release` calls could have led to to the "arbitrarily bad result" of D-Stated or hung tasks in our CF: no doubt the callback to finish up
the last bit of notify mark destruction was on the unlocked structure's `cblist`, which then became corrupted, and thus could never be called. Neat. 

Now I just needed it to happen in one of my envs.

&nbsp;

The next month or so wasn't hugely active. I returned to my normal work and twice a day flicked back to the panes watching for D-States in my three envs,
occasionally chatting to Canonical about my progress.

I got my first repro in my full CF environment after 5 days, and in my lighter envs a couple of days before that. I (sort of) used the [`crash` live kernel analysis tool](http://man7.org/linux/man-pages/man8/crash.8.html)
to mimic how Krein debugged his occurrence. This was the first I had attempted to use Crash, and would have really gone in circles had
Krein not very helpfully pasted and annotated his output. I was able to then eyeball the references which seemed to follow a similar pattern to his
and very, very tentatively judge that ours was indeed The Same Thing. (I have since learned more about Crash, and once I feel I have a solid grip on things
will do a write up because, for love or money, I cannot find a decent tutorial anywhere! If anyone knows of a _good_, _clear_ one which does exist, please email me!)

After reproducing a couple more times (I like to be cautious and pessimistic), I ditched my third env and bumped the kernels on the CF and the first light one
to the test PPA. Given it could sometimes take days to hit that race,
knowing for sure that it was fixed was tricky. I settled on 2 full weeks of a clean run, and threw all my test programs at both environments,
running multiple instances of each in while loops. Then I sat back and listened to my desktop whir louder.

Two weeks later, and it was still making a hell of a din, which meant nothing had become blocked on anything. The full Cloud Foundry
was also still in a healthy state.

And that was it, job done. I reported to Canonical that the patch fixed our problem and could we have that in an image soon please, and by mid April
it was.

&nbsp;

I cannot stress enough that this was an easy and quick thing to "solve" for us. This sort of speed and things "falling into place" almost never happens. From first reporting to proving that it
was the SRCU race giving us grief took a mere 3 months, from there to thoroughly testing the patch to seeing the new kernel in our OS images it was just another 4 months.
It was an extremely open-and-shut case, with the least amount of work on our part, so I apologise if you thought this piece would detail an in-depth investigation; that is
not why I wrote it.

I wrote it because 80% of D-State or other weird kernel problems go unsolved, even with excellent help from folks like Canonical, we were ecstatic that this had
a resolution. We bragged to anyone who would listen. The _other_ D-State we were simultaneously investigating suddenly looked less bleak.

Hopefully this has been entertaining, or perhaps instructive to those looking to extend their DevOps toolset. If you would like to have a go at debugging this problem yourself
all my reproduction materials can be found [here](https://github.com/Callisto13/srcu-race-repro).

&nbsp;

----------------------------------

&nbsp;

_Garden Engineers at time of investigation: [Tom Godkin](https://github.com/BooleanCat) (Pivotal), [Giuseppe Capizzi](https://github.com/gcapizzi) (Pivotal),
[Julia Nedialkova](https://github.com/yulianedyalkova) (SAP), [Georgi Sabev](https://github.com/georgethebeatle) (SAP), [Danail Branekov](https://github.com/danail-branekov) (SAP)
and Claudia Beresford (Pivotal). Product Manager: [Julz Friedman](https://github.com/julz) (IBM)._
