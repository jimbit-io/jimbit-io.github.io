---
title: "Use PowerShell to monitor Windows Defender firewall"
layout: post
category: blog
tags: powershell,windows-server
comments: true
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
{% highlight posh lineos %}
<#
.LINK
  https://learn.microsoft.com/en-us/powershell/module/netsecurity/set-netfirewallprofile
.SYNOPSIS
  Excerpt Windows Defender firewall log remotely.
.DESCRIPTION
  Start Windows Defender firewall logfile by custom name.
  Stop logging after key input.
  Convert standard logfile to usable .csv file.
  Copy file from remote to local location.
.EXAMPLE
  .\ExcerptWindowsFirewallLog.ps1
.NOTES
  OS requirement(s):		- PowerShell Core 7.x (confirmed)
  Module requirement(s)		- No additional modules/features required

  Author:			Jimmy van Ameyde
  Website:			jimbit.io
  Contact:			jva@jimbit.io
				github.com/jimbit-io
				linkedin.com/in/jvameyde/
#>

# Variables (set these by choice)
$remoteHost = New-PSSession -HostName "<remotehost>" -UserName "<username>" -Port 22 -SSHTransport
$fileName = "LogExcerpt"
$fileDir = "C:\fwlogs\" # This directory has to exist when script is run!
$fullLog = "$($fileDir)$($fileName).log"
$fullCsv = "$($fileDir)$($fileName).csv"
$toLocalFile = "/Users/<username>/<subfolder>/$($fileName).csv" # Change to local directory of choice file output
$logWaitTime = 30 # In seconds


# Firewall settings splat for logging everything
$enableFwLog = @{
	Name				= "Domain"
	LogMaxSizeKiloBytes		= "4096" # Default: 4096. When size limit is reached, 1x .old and regular logfile are kept
	LogAllowed			= "True"
	LogBlocked			= "True"
	LogIgnored			= "True"
	LogFileName 			= $fullLog # Default value: "%systemroot%\system32\LogFiles\Firewall\pfirewall.log"
}

# Start Windows Firewall logging on 'Domain' profile
Invoke-Command -Session $remoteHost -ScriptBlock {
	$fwlDomainEnabled = Get-NetFireWallProfile -Name Domain
	if ([string]$fwlDomainEnabled.Enabled -eq 'True') {
		Set-NetFireWallProfile @Using:enableFwLog
		Write-Host "Firewall logging set for profile: Domain"
	}
}


# Wait for set amount before stopping logfiles
Get-Date; Start-Sleep -Seconds $logWaitTime; Get-Date


# Stop Windows Firewall logging on 'Domain' profile
Invoke-Command -Session $remoteHost -ScriptBlock {
	$fwlDomainEnabled = Get-NetFireWallProfile -Name Domain
	if ([string]$fwlDomainEnabled.Enabled -eq 'True') {
		$disableFwLog = @{
			Name				= "Domain"
			LogAllowed			= "False"
			LogBlocked			= "False"
			LogIgnored			= "False"
		}
		Set-NetFireWallProfile @disableFwLog
		Write-Host "Firewall logging stopped for profile: Domain"
	}
}


# Modify generated logfile to useable .csv file
Invoke-Command -Session $remoteHost -ScriptBlock {
	# Copy logfile and rename from .log to .csv extension to avoid Base Filtering Engine file-lock issue
	Copy-Item -Path "$Using:fullLog" -Destination "$Using:fullCsv"

	# Remove first 5 lines (header) from copied .csv file
	Set-Content -Path $Using:fullCsv -Value (Get-Content $Using:fullCsv | Select-Object -Skip 5)

	# Add new header to modify log as useable .csv file, replaces spaces with delimiter of choice
	$csvHeader = "date time action protocol src-ip dst-ip src-port dst-port size tcpflags tcpsyn tcpack tcpwin icmptype icmpcode info path"
	$delimiter = ","
	@($($csvHeader)) + (Get-Content $Using:fullCsv) | Set-Content -Path $Using:fullCsv
	(Get-Content $Using:fullCsv).Replace(" ","$($delimiter)") | Set-Content -Path $Using:fullCsv
}


# Copy remote file to local system
Copy-Item -Path $fullCsv -Destination $toLocalFile -FromSession $remoteHost


# Close remote PowerShell session
$remoteHost | Remove-PSSession
{% endhighlight %}

So what's going on?
Let's start off with that the script invokes commands remotely via SSH remoting. I prefer remoting over SSH because it is cross-platform. This can easily be changed to WinRM style remoting by changing the New-PSSession parameters.

Here's a short overview of its actions;
- There's the variables/splat that need to be checked and set to whatever is needed for the situation. Please note that the default firewall logging directory is: "%systemroot%\system32\LogFiles\Firewall\pfirewall.log". I recommend changing this since having the splat have %systemroot% in its variables does not work well with the rest of the script.
- Section one invokes a simple if-check to see if the 'Domain' profile is set to enabled. If that's the case, the script proceeds to start logging (assuming firewall logging is not already enabled).
- There is variable wait period built-in. During this wait period you can trigger whatever network traffic you're trying to catch (such as in my example: incoming TFTP traffic.
- The firewall logging is then disabled/stopped. This section can of course be commented out, in case you don't want to stop logging.
- The generated logfile is copied to a similar filename but with .csv extension. This also prevents any occuring Base Filtering Engine file-lock errors when trying to modify the original logfile.
- Then lastly, the remote CSV-file gets copied over to a local directory of choice, where it can be imported to Excel or any other CSV viewer/editor.

Now let's see... What do we have here? Looks like we found the culprit.
	
...

> **Disclaimer:** *I try to keep my scripts here relatively simple, if you like the idea but prefer to implement advanced functionality such as: functions, try/catch statements, debugging, etcetera. Feel free to copy the source and change it to whatever is desired. Make sure to understand what is happening before running a script, adjusting it for your situation when and where required.*
