---
title: "Use PowerShell to monitor Windows Defender firewall"
layout: post
category: blog
tags: powershell,windows-server
---

I'm trying to think of a proper reason for my first technical post to be about Windows Defender firewall, since I don't actually consult it that much. With (hardware) network appliances, providing more than capable monitoring/analyzers, or even old-school switch ACL's, why go through the trouble of using Windows Defender firewall logs which are neither viewable in real-time nor formatted in a proper fileformat?

Well... Most of the time there probably isn't a very good reason to. However, it does offer a few advantages;
- You can't get much closer to the source than the OS logging itself, right? At this level you'll be sure to see if a packet even reaches the system and if it does; how is it processed?
- A lot of (enterprise) networks still have their VLAN gateways reside on switch virtual interfaces, sometimes even without ACLs which make it unfriendly to monitor inter-VLAN traffic.
- WireShark will always be a much better choice to monitor incoming or outgoing traffic within an OS'es NIC. However, most sysadmins do not wish to install 3rd party tools on their VM's. Especially tools such as WireShark which can have a negative effect on reliability and performance within a production environment.

As a real-life example, I wanted to backup a WLC's config over TFTP, which defaults to UDP/69. The file failed to upload to the TFTP server and seeing that both hosts resided within the same subnet, chances were that Defender firewall blocked the incoming traffic.

So let's confirm the theory, right? For this situation I created a PowerShell script which makes it easier to go through the logs...
<!--more-->
# The script
...

So what's going on?
Let's start off with that the script invokes commands remotely via SSH remoting. I prefer remoting over SSH because it is cross-platform. This can easily be changed to WinRM style remoting by changing the New-PSSession parameters.

Here's a short overview of its actions;
- There's the variables/splat that need to be checked and set to whatever is needed for the situation. Please note that the default firewall logging directory is: "%systemroot%\system32\LogFiles\Firewall\pfirewall.log". I recommend changing this since having the splat have %systemroot% in its variables does not work well with the rest of the script.
- Section one invokes a simple if-check to see if the 'Domain' profile is set to enabled. If that's the case, the script proceeds to start logging (assuming firewall logging is not already enabled).
- There is a built in wait period where user-interaction (press Enter key) is required. During this wait period you can trigger whatever network traffic you're trying to catch before continueing.
- The firewall logging is then disabled/stopped. This section can of course be commented out, in case you don't want to stop logging.
- The generated logfile is copied to a similar filename but with .csv extension. This also prevents any occuring Base Filtering Engine file-lock errors when trying to modify the original logfile.
- Then lastly, the remote CSV-file gets copied over to a local directory of choice, where it can be imported to Excel or any other CSV viewer/editor.

> **Disclaimer:** *I try to keep my scripts here relatively simple, if you like the idea but prefer to implement more advanced function, catch statements, debugging, etcetera. Feel free to copy the source and change it to whatever is desired. Make sure to understand what is happening before running a script, adjusting it for your situation when and where required.*
