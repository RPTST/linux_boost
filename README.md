# Proxmox and Debian boost

# How to check if boost is working

    #cat /proc/cpuinfo 
    ...
    cpu MHz         : 2800.548  
    ...

    $lscpu
    Architecture:        x86_64
    CPU op-mode(s):      32-bit, 64-bit
    Byte Order:          Little Endian
    Address sizes:       43 bits physical, 48 bits virtual
    CPU(s):              16
    On-line CPU(s) list: 0-15
    Thread(s) per core:  2
    Core(s) per socket:  8
    Socket(s):           1
    NUMA node(s):        1
    Vendor ID:           AuthenticAMD
    CPU family:          23
    Model:               8
    Model name:          AMD Ryzen 7 1700 Eight-Core Processor
    Stepping:            2
    CPU MHz:             2399.280
    CPU max MHz:         3200.0000
    CPU min MHz:         1550.0000
    BogoMIPS:            6387.26
    Virtualization:      AMD-V
    L1d cache:           32K
    L1i cache:           64K
    L2 cache:            512K
    L3 cache:            8192K
    NUMA node0 CPU(s):   0-15
    Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx mmxext fxsr_opt pdpe1gb rdtscp lm constant_tsc rep_good nopl nonstop_tsc cpuid extd_apicid aperfmperf pni pclmulqdq monitor ssse3 fma cx16 sse4_1 sse4_2 movbe popcnt aes xsave avx f16c rdrand lahf_lm cmp_legacy svm extapic cr8_legacy abm sse4a misalignsse 3dnowprefetch osvw skinit wdt tce topoext perfctr_core perfctr_nb bpext perfctr_llc mwaitx cpb hw_pstate sme ssbd sev ibpb vmmcall fsgsbase bmi1 avx2 smep bmi2 rdseed adx smap clflushopt sha_ni xsaveopt xsavec xgetbv1 xsaves clzero irperf xsaveerptr arat npt lbrv svm_lock nrip_save tsc_scale vmcb_clean flushbyasid decodeassists pausefilter pfthreshold avic v_vmsave_vmload vgif overflow_recov succor smca

    $uname -a
    Linux pi 5.8.0-1016-linux #20-Ubuntu SMP PREEMPT Tue Jan 16 16:16:16 UTC 2021 x86_64 GNU/Linux
    root@pi# cat /sys/devices/system/cpu/cpufreq/boost
    0
    
# Well, let's install some utilities to help us...
    
    sudo apt install linux-cpupower cpufrequtils 

# Command to verify what the system is doing
Output on my system (one block of output like this for each core)

    $sudo cpupower -c all frequency-set -g performance
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

Make sure that you have “performance” or “ondemand” and “Boost > Active: Yes” in your output, like the above. Boost was “No” on my system.  Now, the next part, is we need to make this persist across a reboot. We’ll make a custom systemd service.

Check that cpufrequtils is installed and that it should have it’s own systemd service, run the following command:

    $systemctl status cpufrequtils
    cpufrequtils.service - LSB: set CPUFreq kernel parameters
       Loaded: loaded (/etc/init.d/cpufrequtils; generated)
       Active: active (exited) since Sat 2021-01-16 16:22:30 EST; 41min ago
         Docs: man:systemd-sysv-generator(8)
        Tasks: 0 (limit: 7372)
       Memory: 0B
       CGroup: /system.slice/cpufrequtils.service
       
       Jan 16 16:16:16 pi systemd[1]: Starting LSB: set CPUFreq kernel parameters...
       Jan 16 16:16:16 pi cpufrequtils[38101]: CPUFreq Utilities: Setting ondemand CPUFreq governor...CPU0...CPU1...CPU2...CPU3...CPU4...
       Jan 16 16:16:16 pi systemd[1]: Started LSB: set CPUFreq kernel parameters.

That looks reasonable! It’s set to ondemand – let’s change to performance

# Edit the tools: 

First you can use the Phoronix Test Suite to do performance testing before/after, too, to confirm performance uplift.

    $vi /etc/init.d/cpufrequtils

Let's see what has changed

    $cat /proc/cpuinfo 
    ...
    cpu MHz         : 3322.548  
    ...

That’s a nice bump over the previous cap of 2.8Ghz! And it’s important to understand. This is not an overclock. This is literally how it was designed to work 24/7

This is a lot of words. I’m sorry for that. The governor is set, but not the boost. If you use command from a while back in this guide to check boost after a reboot, the boost is no longer boosting.

I don’t know of a more elegant way to make that stick other than creating a custom systemd service. I’m so, so sorry for that.
Creating a systemd service to enable turbo boost

Create a script to enable turbo (this is not strictly necessary since we’re just running one command HOWEVER you’ll thank me if you end up using this service to dump other tweaks that disappear on reboot and don’t have another more elegant spot that they can live.)

# Setup script

Create a file at /usr/local/bin/ with these contents and chmod +x /usr/local/bin/enable-turbo.sh to make it executable.

    $sudo nano /usr/local/bin/enable-turbo.sh

    #!/bin/sh
    echo 1 >  /sys/devices/system/cpu/cpufreq/boost

Create a file at /etc/systemd/system/enable-turbo.service
    
    $sudo nano /etc/systemd/system/enable-turbo.service
    
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

    $systemctl daemon-reload
    $systemctl enable turbo-boost

After rebooting and reconnecting you can issue:
    
    $cat /sys/devices/system/cpu/cpufreq/boost 
    
This will verify boost is working. If it is working it should return a 1 for enabled, or 0 for disabled.

    1
