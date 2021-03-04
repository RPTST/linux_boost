# linux_boost

Proxmox Configuration Tweaks

The first egregious thing I encountered was that boost frequencies on the cpu were not enabled. Whisky-Tapdancing-Tango-Foxtrot, system builders?

uname -a
Linux pm9 5.4.78-2-pve #1 SMP PVE 5.4.78-2 (Thu, 03 Dec 2020 14:26:17 +0100) x86_64 GNU/Linux
root@pm9:/etc/default# cat /sys/devices/system/cpu/cpufreq/boost
0

This is nothing specific to proxmox. Many developers working on distros simply do not properly understand that boost frequencies are a part of every modern processor, supported and should be enabled by default. Yet, here we see, they aren’t. That’s one of the reasons I like installing the cpufreq GNOME extension so you can see if the distro has some hilariously silly default setup that’s killing your performance (looking at you, irqbalance).


# Well, let's install some utilities to help us...
apt install linux-cpupower cpufrequtils 

There is also the question of the performance governor. Most people will prefer to run the ondemand performance governor. It’s reasonable and works well. The problem with it, I have found, is that sometimes if your workload is bursty the CPUs will sleep at inopportune times and slow things down. If you can afford the extra electricity cost performance performance governor is nice.

sudo cpupower -c all frequency-set -g performance

Output on my system (one block of output like this for each core)

analyzing CPU 0:
  driver: acpi-cpufreq
  CPUs which run at the same hardware frequency: 0
  CPUs which need to have their frequency coordinated by software: 0
  maximum transition latency:  Cannot determine or is not supported.
  hardware limits: 1.50 GHz - 2.80 GHz
  available frequency steps:  2.80 GHz, 2.40 GHz, 1.50 GHz
  available cpufreq governors: conservative ondemand userspace powersave performance schedutil
  current policy: frequency should be within 1.50 GHz and 2.80 GHz.
                  The governor "performance" may decide which speed to use
                  within this range.
  current CPU frequency: 2.80 GHz (asserted by call to hardware)
  boost state support:
    Supported: yes
    Active: yes
    Boost States: 0
    Total States: 3
    Pstate-P0:  2800MHz
    Pstate-P1:  2400MHz
    Pstate-P2:  1500MHz

Make sure that you have “performance” or “ondemand” and “Boost > Active: Yes” in your output, like the above. Boost was “No” on my system. What a sad panda that makes me!

Now, the next part, is we need to make this persist across a reboot. We’ll make a custom systemd service.

First, because we installed cpufrequtils it should have it’s own systemd service now:

# systemctl status cpufrequtils

cpufrequtils.service - LSB: set CPUFreq kernel parameters
   Loaded: loaded (/etc/init.d/cpufrequtils; generated)
   Active: active (exited) since Sat 2021-01-16 16:22:30 EST; 41min ago
     Docs: man:systemd-sysv-generator(8)
    Tasks: 0 (limit: 7372)
   Memory: 0B
   CGroup: /system.slice/cpufrequtils.service

Jan 16 16:22:30 pm9 systemd[1]: Starting LSB: set CPUFreq kernel parameters...
Jan 16 16:22:30 pm9 cpufrequtils[38101]: CPUFreq Utilities: Setting ondemand CPUFreq governor...CPU0...CPU1...CPU2...CPU3...CPU4...CPU5...CP
Jan 16 16:22:30 pm9 systemd[1]: Started LSB: set CPUFreq kernel parameters.


That looks reasonable! Ofc it’s set to ondemand – let’s change to performance :slight_smile:

Edit: vi /etc/init.d/cpufrequtils
(*This is a bit anachronistic because… init.d … that’s what came before systemd! It’s not really the init system. And yet the systemd service calls this script! *

Zip down to the governor line and change ondemand to performance if that’s your preference. Ondemand is “fine” I just want the extra performance. Mostly you don’t really need to do this, unless you specifically know you’re on one of those edge cases where ondemand does weird stuff.

You can use the Phoronix Test Suite to do performance testing before/after, too, to confirm perf uplift.

I would hope you’re wondering about something from the above cpufreq output:
available frequency steps: 2.80 GHz, 2.40 GHz, 1.50 GHz

Even though Boost: Yes is showing, it’s still saying it tops out at 2.8ghz not 3.35ghz. What gives? It’s just how things are shown with this tool. If you run something in another terminal and re-run to check the frequency, you’ll see higher frequencies on at least some of the cores.

# cat /proc/cpuinfo 
...
cpu MHz         : 3322.548  
...

That’s a nice bump over the previous cap of 2.8Ghz! And it’s important to understand. This is not an overclock. This is literally how it was designed to work 24/7

This is a lot of words. I’m sorry for that. The governor is set, but not the boost. If you use command from a while back in this guide to check boost after a reboot, the boost is no longer boosting.

I don’t know of a more elegant way to make that stick other than creating a custom systemd service. I’m so, so sorry for that.
Creating a systemd service to enable turbo boost

Create a script to enable turbo (this is not strictly necessary since we’re just running one command HOWEVER you’ll thank me if you end up using this service to dump other tweaks that disappear on reboot and don’t have another more elegant spot that they can live.)

Create a file at /usr/local/bin/enable-turbo.sh with these contents and chmod +x /usr/local/bin/enable-turbo.sh to make it executable.

#!/bin/sh
echo 1 >  /sys/devices/system/cpu/cpufreq/boost

Create a file at /etc/systemd/system/enable-turbo.service with this contents

[Unit]
Description=Enable CPU Turbo Boost
After=network.target
StartLimitIntervalSec=0

[Service]
Type=oneshot
ExecStart=/usr/local/bin/enable-turbo.sh


[Install]
WantedBy=multi-user.target

Reload systemd, enable the service, and reboot:

systemctl daemon-reload

systemctl enable turbo-boost

No errors with that, hopefully?! :smiley:

After rebooting and reconnecting to the Proxmox console, you can issue cat /sys/devices/system/cpu/cpufreq/boost to verify boost is working. It should be 1 for enabled, or 0 for disabled.

Man, all those words to get a reasonable out-of-box default. Truly, I am sorry. But it’s fixed forever and should survive system upgrades for many years which should be some consolation.
