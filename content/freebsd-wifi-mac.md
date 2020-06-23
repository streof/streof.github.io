+++
title = "First FreeBSD experience: something with Wi-Fi and Apple hardware"
date = 2020-06-23
[taxonomies]
tags = ["freebsd", "drivers"]
+++

Asking *What is the Wi-Fi password?* gets usually the job done. You're connected
to the Internet and ready to watch the latest race of
[Jelle's Marble Run](https://www.youtube.com/channel/UCYJdpnjuSWVOLgGT9fIzL0g/featured).
But what if your operating system does not know how to control your wireless adapter?
<!-- more -->

# Some background 
When looking back at the history of operating systems, I've always found the BSD
family quite fascinating. Today, BSD's influence is still reflected in many
operating systems including the open-source FreeBSD and NetBSD, but also in MacOS.
A large part of the MacOS userland experience on the command line is BSD. Everytime
you use `cd`, `grep`, `ifconfig`, `ls`, `ssh`, etc on your Mac, you're using
a BSD program (run for example `man cd` to see for yourself).

*But wait, why FreeBSD in particular? They are many other BSD variants out
there.* My short answer would be beginner friendliness: FreeBSD has a large
community and well-written documentation.

So let's move on. I've got FreeBSD installed on a Macbook Pro (Mid 2015)
running macOS Mojave and the Wi-Fi isn't working, what do I do next? Well,
2 things came into my mind: 

- I need to know more about the built-in Wi-Fi on my laptop.
- Does FreeBSD support this chip? In other words, can I somehow install a program 
(i.e. device driver) that will allow FreeBSD to control my Wi-Fi?

# System Information
On my laptop, I went to `System Information` and under `Network -> Wi-Fi` I found
the Ethernet interface `en0`. The first 2 lines are:

```
Card Type:	        AirPort Extreme
Firmware Version:	Broadcom BCM43xx
```

That's pretty valuable information already. So I went on to see if I can find
some information about FreeBSD drivers support.

When trying to look up general information about Broadcom drivers support on
FreeBSD you might get quite confusing results. The most reliable information on
the subject is probably provided by [Landon Fuller](https://landonf.org/), who
has been working on *improving FreeBSD support for Broadcom Wi-Fi devices*.
Another more general source on open-source wireless devices can be
[found on Wikipedia](https://en.wikipedia.org/wiki/Comparison_of_open-source_wireless_drivers).

But wait, what was one the reasons to choose FreeBSD in the first place?
Exactly, its documentation! From my very limited experience, I can highly
recommend to just stick to the documentation as much as possible at the
beginning of your journey. So let's see, it looks like
[Section 8.3](https://www.freebsd.org/doc/handbook/kernelconfig-devices.html)
from the Handbook is exactly what we need. We already know that our vendor is
Broadcom so we can check if we can find a manual page about it. Luckily there
are [a few man pages](https://www.freebsd.org/cgi/man.cgi?query=Broadcom&apropos=1&sektion=1&manpath=FreeBSD+12.1-RELEASE+and+Ports&arch=default&format=html) where our vendor is mentioned.
When we scroll through the list we see that
[bwn(4)](https://www.freebsd.org/cgi/man.cgi?query=bwn&sektion=4&apropos=0&manpath=FreeBSD+12.1-RELEASE+and+Ports) and 
[bwi(4)](https://www.freebsd.org/cgi/man.cgi?query=bwi&sektion=4&apropos=0&manpath=FreeBSD+12.1-RELEASE+and+Ports)
might be exactly what we are looking for. Cool, we're already making quite
some progress!

# Ports and Packages
After reading through the manual pages, it turns out that the difference between
`bwn(4)` and `bwi(4)` is that the former uses a newer (i.e. v4) version of the
Broadcom's firmware whereas the letter uses v3. Furthermore, both require
an addinational port to be installed (`ports/net/bwn-firmware-kmod` or
`ports/net/bwi-firmware-kmod`, respectively).

At this point, we don't really need to know which driver we'll need, but
what we do want to know is what ports are and how to install one without
direct Internet access. Again the Handbook is the perfect source for this
kind of information. Scanning through
[Chapter 4](https://www.freebsd.org/doc/handbook/ports.html) we learn that
a port is a set of Makefiles, patches and description files which has to be
compiled first, whereas a package is already pre-compiled and available as a
binary. 

So can we maybe get `bwn-firmware-kmod` port as a package? A quick search on
[FreshPorts](https://www.freshports.org/net/bwn-firmware-kmod/)
reveals that there is no package available of this port; reason: *this is a
modified version of a restricted firmware*. We also learn that this port
*requires kernel source files in /usr/src* and has 1 build dependency called
[`b43-fwcutter`](https://www.freshports.org/sysutils/b43-fwcutter/) which
is a firmware extractor.

# Build system
To get a better understanding of the build system, let's take a look at the
[Makefile](https://github.com/freebsd/freebsd-ports/blob/master/net/bwn-firmware-kmod/Makefile)
for `bwn-firmware-kmod`. As always the Handbook will help us to understand
what's going on exactly. In this case we need to read 
[Section 5.4](https://www.freebsd.org/doc/en/books/porters-handbook/makefile-distfiles.html).
In short, when building the port the compiler will
1. Try to find 3 `DISTFILES` inside a local `DISTDIR` (which defaults to
`/usr/ports/distfiles`) or otherwise try to download them from one of the
provided `MASTER_SITES`.
2. Then it will use the `b43-fwcutter` to extract the firmware from the drivers
3. The final step is to add some other firmwares and build 3 kernel modules

Looks pretty straightforward so far, but we are not there yet. The build will
need some additional files and settings such as requirements declared by the
[`USES` macro](https://www.freebsd.org/doc/en/books/porters-handbook/uses.html).
In this case its values are `kmod` and `uidfix`. Moreover, every Makefile
[must end](https://www.freebsd.org/doc/en/books/porters-handbook/dads-after-port-mk.html)
with [`bsd.port.mk`](https://github.com/freebsd/freebsd-ports/blob/master/Mk/bsd.port.mk)
which also requires a bunch of other files for licensing, checksums, patching, etc.

To make your life easier I've created a [small shell script](https://gist.github.com/streof/79b6e92d652336e53bb69d48dca81a4b)
that will download the bare minimum you need to successfully build `bwn(4)` and `bwi(4)`
without relying on an Internet connection. The shell script will create and
populate the following directories under `usr/ports`:

```
├── Keywords
├── Mk
│   ├── Scripts
│   ├── Uses
├── Templates
├── devel
│   └── gmake
├── distfiles
├── net
│   ├── bwi-firmware-kmod
│   └── bwn-firmware-kmod
└── sysutils
    └── b43-fwcutter
```

In additional the script will also download the kernel source code and place it
under `usr/src`. See 
[Section 23.5.3](https://www.freebsd.org/doc/en_US.ISO8859-1/books/handbook/makeworld.html#updating-src-obtaining-src)
and [Appendix A.3.5](https://www.freebsd.org/doc/handbook/svn.html#svn-usage)
for more details about the source code. Make sure to download the correct version, otherwise
you'll get a *version mismatch* error when trying to load the module later on.

*Note: An alternative approach would be to download the entire ports collection instead
of cherry-picking.*

# Kernel modules
Once we have all dependencies we can build our ports! But wait, first we need to
transfer all these files to our offline FreeBSD installation. Yeah that's right.
Probably the easiest way is to use an USB flash drive. When partitioning
we need to make sure that FreeBSD will recognize our flash drive. A safe choice would be
the Master Boot Record partiion scheme and a FAT format such as FAT32. Depending on your FreeBSD
settings the OS might try to automatically mount your flash drive. You can run for example
`gpart status` to see if your device has been recognized.
[Section 3.6](https://www.freebsd.org/doc/en/books/handbook/disk-organization.html)
provides more information on how disks are organized in FreeBSD. In my case, I can unmount my USD
drive with `umount /dev/da0s1` and then mount it back again with `mount -t msdosfs /dev/da0s1 media`
to the `media` directory.

Finally we can build our ports! Let's start with `bwn(4)` (the same steps also apply to `bwi(4)`).
According to the ports website we should use `make install clean` to install the port,
so let's do that. If the build was successful `ls /boot/modules | grep bwn` should show
the three freshly built kernel modules! Now we can add the following lines to `/boot/loader.conf`

```
if_bwn_load="YES"      # loaded from /boot/kernel
if_v4_ucode_load="YES" # replace by if_v4_lp_ucode_load="YES" for low power PHY
```

After rebooting our machine (e.g. `shutdown -r now`), run `kldstat` to check if the
kernel modules have indeed been loaded. If something went wrong, you might try to load
them manually using `kldload` and check the output of `dmesg` for some useful debug information.


Great, we've successfully built our first kernel modules! The next step is crucial and very
exciting. Run `sysctl net.wlan.devices` to see if the kernel can operate our Wi-Fi
device using the driver(s) we just installed. Depending on which driver you used the output
should probably be either `bwn0` or `bwi0`. Cool, this was the hardest part. The rest is
just a matter of Wi-Fi configurations. There are many resources out there which you can follow.
One of them is [this YouTube video](https://www.youtube.com/watch?v=nasH0VLkqrY) by RoboNuggie.
Small remark regarding the video: you might want to use `service network restart` instead of
`service netif restart`.

# Does my Wi-Fi finally work?
Unfortunately, for me the answer was still *No*. It turned out that on my machine `BCM43xx` stands
for `BCM4360` (or to be more specific: `BCM43602`) which is a quite modern chipset and neither
`bwn(4)` nor `bwi(4)` supports it. According to Landon Fuller, FreeBSD
[might support](https://landonf.org/code/freebsd/Broadcom_WiFi_Improvements.20180122.html)
these newer chipsets in the future.

*Note: OpenBSD and NetBSD [already](https://man.openbsd.org/bwfm) [support](https://netbsd.gw.com/cgi-bin/man-cgi?bwfm++NetBSD-current) BCM43602.*

# Easy way out
After running out of options, I was forced to use the Ralink wireless adapter from my Raspberry Pi.
Luckily, the RT5370 chipset is supported by the `run(4)` driver which is part of the FreeBSD kernel.
So adding the following two lines to `/boot/loader.conf` sealed the deal:

```
if_run_load="YES"
runfw_load="YES"
```

Now I can finally watch the [latest race](https://www.youtube.com/watch?v=pt5DHwyEL_E)
of Jelle's Marble Run on FreeBSD! 

