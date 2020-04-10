---
layout: post
permalink: /blog/remote-pair-programming
title: "How to pair-program remotely and not lose the will to live"
date: "April 10, 2020"
---

### _Or: there has never been a better time to learn Vim_

Yesterday evening, before wrapping up for the day, I checked in with a member of my team.
We were onboarding a transfer from another team, so I asked the veteran to do some
pair-programming with the new member to get them up to speed.

The response was:

> we did the work on \<ticket\> on vscode (bleh) liveshare

I could immediately and deeply feel the pain in this innocuous response.

That "bleh" is a multitude of frustrations boiled down into a single, tired shrug. "Bleh".

And VS Live Share is actually not terrible! For folks whose natural territory isn't
the command line, who like a GUI, Microsoft have really done a great job at making
collaboration between devs a smoother experience than the one afforded by screen-sharing.

But it still isn't perfect. For one, I am not a fan of the teeny-tiny (non-expandable) terminal
it gives you. How on earth can you read logs in that?
And even though they have got the lag down to a pretty acceptable level,
this is really only tolerable if you are jumping in for a quick run-through of something.

If you are settling in for a full day of pairing, or if a bug is particularly gnarly,
then the person who is on the remote end of the share is in for a frustrating and wearing
time: it is always abundantly clear that you are in someone else's "home", and that you lack
the full control and comfort of being in your own space.

And if you are collaborating over screen-share... god help you.

Pairing or collaborating is draining enough without the frustration of lag or
the pain of viewing a blurred screen either bigger or smaller than yours.
Fortunately, with a little time, effort and the right set of tools, you never have
to worry about these problems again.

&nbsp;

--------------------------------------

&nbsp;

_I'm not going to go into pair-programming and what that means, you can read more
about general pairing practices from many blogs like [this](https://tanzu.vmware.com/content/blog/the-elements-of-highly-effective-pair-programming).
This is more about effective collaboration on anything in general._

_I will say this: pair-programming is always, hands-down, the fastest and best way to level up a new dev,
onboard new team-members, and share information and learnings. It is also utterly exhausting,
and going for it 100% of the time will burn people out. It is also much more of a strain
for neuro-atypical people. Use it wisely._

&nbsp;

--------------------------------------

&nbsp;

There are just 4 tools you need to set up your seamless collaboration environment. 2 have hideous,
but not insurmountable, learning curves. 1 is probably something you use often, if not all the time,
and 1 requires minimal setup which you can then forget about.

_This setup is the cumulative design of multiple teams/people across Pivotal, SAP and IBM, and AFAIK is still used
by my old team :). I have added a little extra sugar._

### Terminal

Everything happens here.

Are you sick of your overblown IDE taking ages to boot?
Is your memory consistently swapping at around 5GB?
Do you sometimes anxiously sit and listen to your machine's whirring increase,
and worry that this may be the time that it finally launches into orbit?
How often do you consider cancelling your heating contract, and simply placing laptops
running IntelliJ throughout your home?

Uninstall it! In this setup, all your work can be done from the command line, shrinking
all your workload footprints down to reasonable levels.

If you have a Mac you can go for the ritzier iTerm2, but the regular pre-installed Terminal
App will work just fine.

Some familiarity with Unix is required, so if this is new to you I would recommend checking out some videos
on YouTube (of which there are many, [this](https://www.youtube.com/watch?v=oxuRxtrO2Ag)
and [this](https://www.youtube.com/watch?v=HjuHHI60s44) seem fairly comprehensive).

### ngrok

Ngrok is a secure tunneling reverse proxy tool. It has many nifty uses, but in our case
this is what is going to let your collaborating buddy securely SSH into your machine and,
eventually, see whatever you see.

Sign up [here](https://ngrok.com/product) for a free account, install the binary
somewhere on your path, follow the instructions to set your auth-token.

_If you have set custom networking and firewall rules on your machine, you may need
to tweak those later in order to get our SSH tunnels working._

### tmux

So, ngrok lets your buddy SSH into your machine, but you are in separate sessions,
you can't yet see or work on the same things.

For that we need tmux. We can use this to create a new session to which all parties can attach.
This is one of the things which come with a learning curve, but once you have it
down it is really awesome and a pretty cool thing to use day-to-day, even when you aren't working with someone remotely.

[This post here](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/) is a really good
intro to installing and getting started with tmux. It also lists more reasons why
putting in the effort to learn it properly is totally worth it.

You can also set your own config (in `~/.tmux.conf`). Mine can be found [here](https://gist.github.com/Callisto13/b4cc217ca4f1c2f7f51405d62b941adb).
Key highlights include enabling mouse clicking, and resetting the leader sequence to `ctrl+space`.
If you want to use my config, you will also need to install a [plugin manager](https://github.com/tmux-plugins/tpm).

### NeoVim

Once we are all in the same terminal session, we need a terminal based editor.
There are a bunch to choose from (nano, emacs, vim, etc) and all come with steep
learning curves. Knowing any terminal editor pays off, however, and the best way
really get it is to use it every day until it becomes muscle memory.

I use NeoVim, which is basically Vim but with a few bells and whistles. You can
start learning Vim by opening up a terminal and typing `vimtutor`. There is then plenty
of stuff online for when you get more advanced.

Bare-bones Vim/NeoVim is pretty feature-limited, so you are definitely going to want to
add some bells and whistles of your own to get that full-bodied IDE experience.
Some folks like to write and maintain their own full vim configs (found in `~/.vimrc`)
and colourschemes.
If this is something you are into, or want to learn, [this blog](https://dougblack.io/words/a-good-vimrc.html)
is a good place to start. You still don't need to do everything yourself, as there
are hundreds of plugins out there for everything from Language Server Protocols to
inline Github integrations. To manage your Plugins you of course need a
[plugin manager](https://www.slant.co/topics/1224/~best-plugin-managers-for-vim)
and, as with everything else, you can find many opinions on which is best on StackOverflow.

It is also perfectly legit to use someone else's (I got bored of maintaining mine after about 6 months).
The [one I use now](https://github.com/masters-of-cats/a-new-hope) is maintained by my old team.
I don't like every configuration option, so I also
have my [own `~/.vimrc`](https://gist.github.com/Callisto13/dc6448d5c6606a18be6ad79b9a22fc07)
and [plugins](https://gist.github.com/Callisto13/436b6496e1b6ef149b88c38e8cc1d1a4)
to override some of their more irritating choices.

_There are zero docs on how to install that config, so [here is a gist](https://gist.github.com/Callisto13/440f21632b75b289d02daa81cf65e6ee) while I PR some.
`a-new-hope` is geared up to write either Go code or shell scripts, but you can add whatever you like to work in
your own language._

&nbsp;

--------------------------------------

&nbsp;

Now that we have all the tools, let's collaborate!

### Fetch some keys

The first step is for the host to add their buddy's public key to their `~/.ssh/authorised_keys`
file. This step is not necessary for someone you trust well enough to share your password,
but you most likely want to keep that to yourself.

Adding their public key will let them SSH into your machine over your ngrok tunnel.
You can either ask them to send you one, or you can grab keys from Github.

Create yourself and dirty little helper script to add their keys to your `~/.ssh/authorized_keys`:
```sh
cat << EOF > ~/get_keys
#!/bin/bash -e

buddy="$1"

keys=$(curl "https://api.github.com/users/$buddy/keys" | jq -r '.[] | .key')

printf "%s\n" "$keys" >> ~/.ssh/authorized_keys
EOF

chmod +x ~/get_keys
```

And run it:
```
./get_keys <your buddy's github name>
```

### Create a new tunnel

The host next needs to open up a secure tunnel via ngrok. (Hopefully when installing
ngrok you verified it was working.)

Let's write another dirty helper script for this:
```sh
cat << EOF > ~/start_tunnel
#!/bin/bash

lsgrok() {
  url=$(curl http://localhost:4040/api/tunnels 2>/dev/null | jq -r .tunnels[0].public_url)

  if [ "$url" == "" ] || [ "$url" == "null" ]; then
   echo No tunnels running?
  else
   host=$(echo "$url" | cut -d / -f 3 | cut -d : -f 1)
   port=$(echo "$url" | cut -d / -f 3 | cut -d : -f 2)

   echo ssh -p "$port" "$(whoami)"@"$host"   >>  Quote this to prevent word splitting.
  fi
}

ngrok_init() {
  tmux new-session -s ngrok -d
  tmux send -t ngrok "ngrok tcp --region=eu 22" ENTER
}

ngrok_init
sleep 1
lsgrok
EOF

chmod +x ~/start_tunnel
```

And call it:

```sh
./start_tunnel
# you should get something like this returned
ssh -p 15556 callisto@0.tcp.eu.ngrok.io
```

Copy the resulting SSH command and send it to your buddy. They should be able to run
the command from their terminal and find themselves in your machine.
You are nearly ready to pair!

### Join a shared session

The last thing to do is fire up a new tmux session with a name that is easily identifiable
to both of you. Since you are both in the same machine either of you can create it,
with the other running the attach command.

One person runs:

```sh
tmux new-session -s seamless-pairing
```

And anyone else wishing to join runs:

```sh
tmux attach -t seamless-pairing
```

Boom! You are now able to work together seamlessly, almost completely lag-free.
Open a terminal editor and get cracking!


&nbsp;

--------------------------------------

&nbsp;

The best sharing setup should make you feel completely in control. When using
these tools together, I have often totally forgotten that I am not using my own computer.
I have spent many embarrassing minutes searching in the `~/Downloads` folder for
something I clicked on in my browser, only to belatedly realise that, of course,
that it ended up on _my_ machine, and not in the filesystem I am looking at.

If you pair regularly, it is a good idea to have a common set of tmux and editor
configs. This will mean that anyone in the team can  move around and write code in any
machine they happen to be working from.

**It goes without saying that you should only give people you trust access to your machine.**

Of course, no system is perfect. Terminal based editors have their own set of
problems and do not provide the idiot-friendly user experience of IDEs out of the box.
It takes a while to fully master enough of something like Vim to realise its power
and not be frustrated by it, especially if you are coming from something which does
everything but write the code for you (um, well, IntelliJ...).
Few people have the patience or discipline for that, not that there is anything
special about me: I had no choice but to learn it as my team was a 100% pair-programming
one and this setup was voted the least terrible of an endlessly disappointing selection.

The downside of tmux is everyone's terminal size is set to that of the smallest screen.
In other words, if you are on a 27" monitor, and your buddy is on a 13" laptop,
your terminal will be laptop-screen sized. For me the control, absence of lag and
clarity of picture is an acceptable trade-off.

I hope this has been useful to some, and that even if the full pairing setup isn't for you,
maybe you picked up some cool new tricks for your workflow.
Yes, Vim and tmux are hard and yes it takes some fiddling to get a great environment
going, but what else have you got to do right now?

&nbsp;

--------------------------------------

&nbsp;

_If you experience any issues and either solve them or need help troubleshooting,
please drop me an email._
