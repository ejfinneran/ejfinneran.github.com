---
layout: post
title: "OpenStack Simple Scheduler"
date: 2011-09-14 20:04
comments: true
categories:  
---

Interesting gotcha found while playing with OpenStack.

We were working on a two node OpenStack cluster but ran into a problem where the scheduler was not properly balancing between the two nodes.  We'd spin up 10 instances and eight would spawn on one box and two on the other.  Not ideal, obviously.

We figured out that you can specify a different scheduler via nova.conf.  We added:
```
--scheduler_driver=nova.scheduler.simple.SimpleScheduler
```

Despite its name, SimpleScheduler tries to intelligently schedule new instances based on the current load of the available compute nodes.  That solved our issue!

This isn't very well documented.  I had to go digging through the OpenStack code to figure out the syntax we needed but that's why open source is awesome. :-)

Overall, OpenStack has blown me away. Awesome stuff.
