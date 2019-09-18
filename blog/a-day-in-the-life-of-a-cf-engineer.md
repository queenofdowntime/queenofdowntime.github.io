---
layout: post
permalink: /blog/a-day-in-the-life-of-a-cf-engineer
title: "A Day in the Life of a Cloud Foundry Engineer"
date: "May 09, 2018"
---

_this post originally appeared at [pivotal.io](https://content.pivotal.io/blog/a-day-in-the-life-of-a-cf-engineer)_

My day begins with me running into a meeting room juuuust in time for my team’s 9:15am stand-up. My team is standing around the desk, facing the tv, and waiting for the remote members to come online. We used to have sit-down stand-ups (standing up feels weird when half your team is in Bulgaria) but we realized that made us lazy and these short catch-up meetings were taking an unreasonable amount of time, so now we are on our feet again.

Even so, it takes a while. This team has recently re-absorbed the spin-off project it spawned a year and a half ago, and is now twice the size. Five pairs need more than 5 minutes to share and reorganize for the coming day. Twenty minutes later we are still discussing which combination of engineers would work best for tackling the many weird and wonderful bugs we have woken up to this morning.

Finally, with pairs decided, we move onto the “Parking Lot”. While organizing priorities and pairs for the day, any team member who has some useful information to share about a particular track of work, but did not want to hold up the pairing process, punts it to the Parking Lot discussion held immediately after. Attendance in the Parking Lot is optional, and I am usually torn between running upstairs to see if there is any breakfast left, and staying to listen to something which is potentially relevant to my interests.

All useful information shared, and a wider-reaching topic pushed to a dedicated meeting some other time, we disperse to get on with our day. I run upstairs to find that most of breakfast has been eaten (I can’t complain; if I wanted food I shouldn't be late), grab a bagel, and return downstairs to claim my desk.

Today I’m paired with a team member in Bulgaria, and move my treasured personal possessions (junk) to a couple of monitors which I arrange into a suitable hacker formation. I plug in my clickiest-clackiest keyboard, put on my favorite headphones, and dial my colleague in over Slack.

Our team develops Garden, the container runtime for [Cloud Foundry](https://www.cloudfoundry.org/application-runtime/). Until quite recently I worked on the component which provided the root filesystems for [Garden containers](https://docs.cloudfoundry.org/concepts/architecture/garden.html), [Grootfs](https://github.com/cloudfoundry/grootfs#grootfs-garden-root-file-system). Since Groot has finished its mandate (sort of), the teams are now combined while we roll out some cool new features, and handle some interesting quirks in Groot which seemed to manifest immediately after we pronounced it production ready. Both codebases are written in Go, and leverage Linux containerisation technologies in a way that is reminiscent of Docker, but more lightweight and dedicated to Cloud Foundry’s use case.

We remote pair using an ngrok ssh session which allows us to share a terminal and use our text editor of choice, vim. As I was working on our track yesterday, I first share context of what has been done so far, and what we have yet to do. We work together until my partner, who is in a different timezone, goes for lunch. I take the opportunity of his absence to deal with some emails, and then carry on with our story. They come back just in time for me to give an update on my progress before I go off for my lunch.

It’s Monday, so after lunch we have IPM (which stands for Iteration Planning Meeting but we are agile so only talk in acronyms and puns). This is a weekly meeting in which our Product  Manager scopes out the coming week's body of work, and the engineers discuss and estimate the complexity of each feature.

Then it’s back to work. My afternoons vary from being uninterrupted (this only happens if there are no production issues at all), to being full of conversations with teams in western timezones who are only now waking up. On Tuesdays, we sometimes schedule an hour to do tech presentations in which anyone can show-and-tell on something interesting they learned recently, or on a tricky bug they solved. Every Friday at 3pm we have a retrospective for the past week which is usually just an opportunity to rant about some piece of technology causing us more pain than it was worth.

My last 2 hours are often spent working alone, if pairing with a ‘Bulgardener’. I use some of this time to summarize the work done so far and ensure that all code is pushed to work-in-progress branches; we try not to leave anything lying around on any specific machine. The rest of the time I spend learning something new, or writing [talks and workshops for conferences](/talks).
Tomorrow I could end up working on the same piece of code, or a different one with a different partner, or someone will discover a new bug (or two) and the team will allocate someone to drop everything to investigate. Either way I am looking forward to it.

