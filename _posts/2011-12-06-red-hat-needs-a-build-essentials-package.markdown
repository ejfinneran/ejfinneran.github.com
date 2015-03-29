---
layout: post
title: "Red Hat needs a build-essentials package"
date: 2011-05-04 23:08
comments: true
categories: 
---
Dear Red Hat,

Please fix this:

```
[root@localhost ~]# yum groupinstall "Development Tools"
....
Transaction Summary
=================
Install 133 Package(s)
Upgrade 9 Package(s)

Total download size: 137 M
```

vs Ubuntu

```
user@ubuntu:~$ sudo apt-get install build-essential
Reading package lists... Done
Building dependency tree 
Reading state information... Done
....
0 upgraded, 16 newly installed, 0 to remove and 0 not upgraded.
Need to get 19.6MB of archives.
After this operation, 66.8MB of additional disk space will be used.


```

19.6MB to your 137MB.

I just want the packages I need to compile and install software, I don't want every development library available on the system.
