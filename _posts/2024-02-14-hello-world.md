---
title: "Hello World"
layout: single
tags:
    - ramblings
---

Hello world from Github Pages. This post has two purposes: 1) get a post on the
main page to give the site some content; 2) introduce the blog.

Every respected developer, security practitioner, person in tech seems to have a
well developed blog. It's not necessarily for a purpose other than to
communicate to the world what they are currently working on, things they have
found out, etc. I'm not claiming to be someone that has that much useful knowledge to
share, but maybe something I'm working on or a problem I solve (with the help of
other blogs) will help someone out along the way. 

I can't begin to lists the countless blog posts that have helped me solve so
many problems. Whether it be an old Hack the Box walkthrough, a deep dive on a
particular Rust feature, or something as simple as converting a VMware image to
qcow2 for use in Proxmox (it's not actually that simple), I owe a lot of my
knowledge to blog posts that fill the internet. I only hope I can assist someone
in the future. 

## What to Expect
I'm currently in the process of "designing" a typical
security tunneling tool in Rust. There are some great tunneling tools out there
like Chisel, sshuttle, ligolo, that I have used before, but I don't think I
have seen any that are written in Rust. There's nothing wrong with the tools I 
listed or the languages they are written in, but there is something wrong with
my skills as a Rust developer. I plan (and hope) to change that with this
project.

I will try to write some blog posts along the way to include my initial design
thoughts, random things I learn (for example, I had no idea that running the
command `ip route` actually used NETLINK sockets to get the information from the
kernel), and the status of the project. I plan to structure the project with
certain milestones so I don't feel overwhelmed with the undertaking. It will
likely take me a long time to get this to a solid point that is usable, but
hopefully some blog post content can help fill in the gap before the tool is
completed. As of right now (subject to change), I want the tool to meet these
two goals: 

1. Easy UI to create tunnels
2. Highly performant (it's Rust after all).

I will also use the blog to post random thoughts (tagged as 'ramblings') and also
post any security related things that I find interesting. 

Hope you enjoy it!

Goodbye for now world.
