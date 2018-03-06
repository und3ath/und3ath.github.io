---
layout: post
title: Synology DSM Unsupported Usb Wifi Dongle Integration
date: 2018-03-06 00:00:00
---

In this first post i will talk about Synology nas, specialy the DS218J i bought recently.  

I want to connect my nas in wifi using my D-link Dwa131 usb wifi dongle.  
Surprisingly he does not recognize my device.  
After googling around it came that the DS218J ony support D-Link DWA131 A1, wich use RTL8192SU chip, while my D-Link DWA131 E1 use a RTL8192EU chip.  
The Synology don't have a driver for that chip.  
I do not want to purchase another dongle so i decided to figure out how i can integrate myself to fully support my hardware.  


First i need to take the linux driver for that chip and cross compil it to be used in the DSM.  
I choose [https://github.com/Mange/rtl8192eu-linux-driver](https://github.com/Mange/rtl8192eu-linux-driver), it's the official one with additional patch to keep it working on newer kernel.  

To cross compile a kernel module for the DSM we need to get the cross compilation toolchain and kernel source provided by synology (thanks).  
{% highlight shell-session %}
root@und3ath-storage:~# uname -a
Linux und3ath-storage 3.10.102 #15254 SMP Fri Jan 26 06:44:33 CST 2018 armv7l GNU/Linux synology_armada38x_ds218j
{% endhighlight %}

According to the uname output above i download the following toolchain:  
[https://sourceforge.net/projects/dsgpl/files/DSM 6.1 Tool Chains/Marvell Armada 38x Linux 3.10.102/](https://sourceforge.net/projects/dsgpl/files/DSM 6.1 Tool Chains/Marvell Armada 38x Linux 3.10.102/)
And the below kernel source:  
[https://sourceforge.net/projects/dsgpl/files/Synology NAS GPL Source/15047branch/armada38x-source/linux-3.10.x-bsp.txz/](https://sourceforge.net/projects/dsgpl/files/Synology NAS GPL Source/15047branch/armada38x-source/linux-3.10.x-bsp.txz/)

Extracted both and moved the compilation toolchain folder under /usr/local  
First set environment variable for cross compilation  
{% highlight bash %}
export CFLAGS="-I/usr/local/arm-unknown-linux-gnueabi/include"
export LDFLAGS="-I/usr/local/arm-unknown-linux-gnueabi/lib"
export RANLIB=/usr/local/arm-unknown-linux-gnueabi/bin/arm-unknown-linux-gnueabi-ranlib
export LD=/usr/local/arm-unknown-linux-gnueabi/bin/arm-unknown-linux-gnueabi-ld
export CC=/usr/local/arm-unknown-linux-gnueabi/bin/arm-unknown-linux-gnueabi-gcc
export LD_LIBRARY_PATH=/usr/local/arm-unknown-linux-gnueabi/lib
export ARCH=arm
export CROSS_COMPILE=/usr/local/arm-unknown-linux-gnueabi/bin/arm-unknown-linux-gnueabi-
export KSRC=/synology/linux-3.10.x-bsp/
{% endhighlight %}

KSRC is the folder where the kernel source where extracted.
After that, copy the appropried kernel conf for your hardware and make dep and modules

{% highlight shell-session %}
root@und3ath-virtual-machine:/synology/linux-3.10.x-bsp# cp synoconfigs/armada38x .config
root@und3ath-virtual-machine:/synology/linux-3.10.x-bsp# make oldconfig
root@und3ath-virtual-machine:/synology/linux-3.10.x-bsp# make dep
root@und3ath-virtual-machine:/synology/linux-3.10.x-bsp# make modules
{% endhighlight %}


So the last step before compiling is to modify the makefile of the driver. 

set
{% highlight bash %}
CONFIG_PLATFORM_I386_PC = y
to
CONFIG_PLATFORM_I386_PC = n
{% endhighlight %}
add a new variable below
{% highlight bash %}
CONFIG_PLATFORM_ARM = y
{% endhighlight %}
and add the condition below
{% highlight bash %}
ifeq ($(CONFIG_PLATFORM_ARM), y)
EXTRA_CFLAGS += -DCONFIG_LITTLE_ENDIAN
ARCH := arm
endif
{% endhighlight %}

Save the makefile and compile the driver 
{% highlight shell-session %}
root@und3ath-virtual-machine:/synology/rtl8192eu-linux-driver# make
{% endhighlight %}

Copie the rtl8192eu.ko file to the /lib/modules/ folder of the nas.  
Load our previously compiled driver with insmod and plug the dongle to confirm that your device is reconized.  

{% highlight shell-session %}
root@und3ath-storage:~# insmod /lib/modules/rtl8192eu.ko
root@und3ath-storage:~# lsmod | grep 8192eu
8192eu                962571  0
usbcore               134272  8 usblp,usb_storage,ehci_hcd,ehci_pci,ehci_orion,usbhid,8192eu,xhci_hcd
root@und3ath-storage:~# iwconfig
wlan0     unassociated  Nickname:"<WIFI@REALTEK>"
          Mode:Managed  Frequency=2.412 GHz  Access Point: Not-Associated
          Sensitivity:0/0
          Retry:off   RTS thr:off   Fragment thr:off
          Encryption key:off
          Power Management:off
          Link Quality:0  Signal level:0  Noise level:0
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:0   Missed beacon:0

sit0      no wireless extensions.

lo        no wireless extensions.

eth0      no wireless extensions.
{% endhighlight %}

So, wifi dongle is corrected reconized and cli configuration of wifi network is functionnal but it not appeare in the DSM Web Gui.  
After little reversing of diverse component, DSM web gui use a file placed in tmp with name pattern of wireless.conf{id} wich describe the usb dongle.  
This file is generated by udev scripts under /lib/udev/scripts wich also initialize wifi deamon and other wireless related configs.  
To accomplish that, we need to add an entry in the usb.wifi.table under /lib/udev/devicetables/ and modify the udev script wich handle usb wifi events.  

By issuing lsusb we get necessary informations:  
{% highlight shell-session %}
root@und3ath-storage:~# lsusb
|__usb1          1d6b:0002:0310 09  2.00  480MBit/s 0mA 1IF  (Linux 3.10.102 xhci-hcd xHCI Host Controller f10f0000.usb3) hub
  |__1-1         2001:3319:0200 00  2.10  480MBit/s 500mA 1IF  (Realtek Wireless N Nano USB Adapter 00e04c000001)
|__usb2          1d6b:0003:0310 09  3.00 5000MBit/s 0mA 1IF  (Linux 3.10.102 xhci-hcd xHCI Host Controller f10f0000.usb3) hub
|__usb3          1d6b:0002:0310 09  2.00  480MBit/s 0mA 1IF  (Linux 3.10.102 xhci-hcd xHCI Host Controller f10f8000.usb3) hub
|__usb4          1d6b:0003:0310 09  3.00 5000MBit/s 0mA 1IF  (Linux 3.10.102 xhci-hcd xHCI Host Controller f10f8000.usb3) hub
|__usb5          1d6b:0002:0310 09  2.00  480MBit/s 0mA 1IF  (ehci_hcd f1058000.usb) hub

{% endhighlight %}


Usb Vendor ID is 2001 and Product ID is 3319   
We need to add an entry to the usb.wifi.table with these info and the driver name.  
{% highlight shell-session %}
echo "(0x2001:0x3319,rtl8192eu)" >> /lib/udev/devicetable/usb.wifi.table
{% endhighlight %}

The last step is to modify the usb-wifi-utils.sh script to support our device.  
At the top of this file add an entry for our device chip.  
{% highlight bash %}
RTL8192EU_MODULES="rtl8192eu" # driver for D-Link dwa131E1g wifi dongle
{% endhighlight %}

Locate the `select_modules` function and add this switch  
{% highlight bash %}
rtl8192eu)
	modules=${RTL8192EU_MODULES}
	;;
{% endhighlight %}

Now plug the usb dongle and Synology Web Gui will reconize your device.  
This procedure can be applied to other unsuported usb wifi dongle if you have the corresponding linux driver source.  

und3ath. 
