---
title: "Use PowerShell to monitor Windows Defender firewall"
layout: post
category: blog
tags: powershell,windows-server
---

I'm trying to think of a proper reason for my first real post to be about Windows Defender firewall, since I hardly ever monitor it. With (hardware) network appliances, providing more than capable monitoring/analyzers, or even old-school switch ACL's, why go through the trouble of using Windows Defender firewall logs which are neither viewable in real-time nor formatted in a proper fileformat?

Well... Most of the time there probably isn't a very good reason to. However, it does offer a few advantages;
- You can't get much closer to the source than the OS logging itself, right? At this level you'll be sure to see if a packet even reaches the system and if it does; how is it processed?
- A lot of (enterprise) networks still have their VLAN gateways reside on switch virtual interfaces, sometimes even without ACLs which make it unfriendly to monitor inter-VLAN traffic.
- WireShark will always be a much better choise to monitor incoming or outgoing traffic within an OS'es NIC. However, most sysadmins do not wish to install 3rd party tools on their VM's. Especially tools such as WireShark which can have a negative effect on reliability and performance within a production environment.

As a real-life example, I wanted to backup a WLC's config over TFTP, which defaults to UDP/69. The file failed to upload to the TFTP server and seeing that both hosts resided within the same subnet, chances were that Defender firewall blocked the incoming traffic.

So let's confirm the theory, right? For this situation I created a PowerShell script which makes it easier to go through the logs...
<!--more-->
# The script laid out
I try to keep my scripts here relatively simple, if you like the idea but prefer to implement more advanced function, catch statements, debugging, etcetera. Feel free to copy the source and change it to whatever is desired.

So what is going on?
Let's start off with that it invoked commands remotely via SSH remoting. I prefer remoting over SSH because it is cross-platform. This means both sides need a minimum of version 6x. Here's a short overview of its actions;
- There's the variables/splat that need to be checked and set to whatever is needed for the situation.
- Section one invokes a simple if-check to see if the 'Domain' profile is set to enabled. If thats the case, the script proceeds to start logging (assuming firewall logging is not already enabled).

...post to be continued soon :)...
