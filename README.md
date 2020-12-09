# ezjail-3.4.2

# Overview
A Jail in FreeBSD-speak is one or more tasks with the same kernel Jail-ID, bound on zero or more IP addresses, having the same chroot-environment. One usecase of the FreeBSD Jail Subsystem is to provide virtual FreeBSD-systems within a Host-system. ezjail is about making this as easy as possible, aiming for minimum system resource usage. All further references to the term Jail are to a virtual FreeBSD-system consisting of a host name, an IP-address and a Jail root.

The jail(8) man page outlines the way to create Jails, however, when you need several Jails, complete Jail Directory Trees quickly use much of your valuable hard disc space. ezjail avoids this by using FreeBSDs nullfs feature. Most of the base system (/bin, /boot, /sbin, /lib, /libexec, /rescue, /usr/{bin, include, lib, libexec, ports, sbin, share, src}) only exists in one copy in the Host-system and is being mounted read only into all Jails via nullfs. Those Jails are quite slim (around 2mb each) and only consist of some soft links into the basejail mount point and non-shared directories like /etc, /usr/local, etc.

The ezjail approach offers lots of advantages:

You save disc space, inodes and even memory since the system only needs to hold one copy of base system binaries for all Jails
You can update all Jails on a single base directory, since it is so eazy, you might actually end up doing it
Intruders compromising Jails are unable to install standard rootkits (as the base system is mounted read only)
Since ezjail is written entirely in sh, there is no need to install other script languages into the hostsystem
As the base system is provided via soft links, the enjailed users can choose not to use the mounted world
ezjail offers full zfs integration and can help you automatize your file system configuration
An often underestimated fact: less complexity means more security.

# Quick start
To set up your first very simple ezjail, just install ezjail from sysutils/ezjail port or via pkg_add -r ezjail and enable it by setting ezjail_enable=YES in your in your /etc/rc.conf. Assuming your network interface is em0, just type (as root):
```
ezjail-admin install
ezjail-admin create example.com 'em0|10.0.0.2'
ezjail-admin start example.com
```
and you're done. Get a shell in your new jail with the:
```
ezjail-admin console example.com
```
command. As with any vanilla FreeBSD installation, you might probably need to touch /etc/ and maybe copy your host's /etc/resolv.conf.

#Slow start
ezjail comes with some sane defaults, but can be configured globally and per jail using the config file /usr/local/etc/ezjail.conf (copy the sample from /usr/local/etc/ezjail.conf.sample) and the per-jail config files under /usr/local/etc/ezjail/ (those are created automatically with the jails and managed by the ezjail-admin config command).

# ZFS
ezjail integrates nicely with zfs, ready to manage all jails in its own file system. So if your system has a zpool configured, tell ezjail to use zfs and which zpool to use for its house keeping:
```
uncomment the ezjail_use_zfs=YES
```
point the ezjail_jailzfs variable to a file system that will be created by ezjail-admin install, (e.g. tank/ezjail)
while you're at it, you can tell ezjail to create all future jails in their own file system (which defaults to be a child of ezjail_jailzfs)
```
uncomment ezjail_use_zfs_for_jails=YES
```
now the commands in the quick start example should set up a zfs hierarchy ready to use all the nifty features of zfs.

# Flavours
ezjail can help you with the otherwise tedious task of decorating the interior of new jails–those come as naked FreeBSD installations by default. A set of files to copy, packages to install and scripts to execute is called "flavour". ezjail comes with an example flavour called "example" that comes pre-tuned for the use in jails, with an appropriate rc.conf, make.conf, periodic.conf and /usr/local/etc/sudoers.

You are encouraged to copy the flavour and modify the contained script to suit your needs–flavours reside in the directory configured with the ezjail_flavours_dir variable, which defaults to /usr/jails/flavours. But just calling:
```
ezjail-admin create -f example example.com 'em0|10.0.0.2'
```
should do. Note, that the flavour script is being run the first time the jail starts, so calling:

```
ezjail-admin console -f example.com
```
is a nice idea. You can use the shell to further configure the new jail.

# The basejail
All jails share a read only mounted copy of the FreeBSD base system, in ezjail this is called basejail. The quick start section gave a glimpse on the most simple way to install just the basics, but no ports tree, no man pages (pre FreeBSD-9) and no sources. You can run the ezjail-admin install command with the options -P, -M and -S again to install these distribution packages without installing the base system again, or just call ezjail-admin install -spm from start. ezjail uses the portsnap command to provide (and later update) the ports tree. If you do not want to install the OS version running in the host system, call:
```
ezjail-admin install -r 2.2.8-RELEASE
```
If you want to install your base system from source, use the ezjail-admin setup command (also called ezjail-admin update); assuming you have already built your world for the host system, you would just call:
```
ezjail-admin setup -i
```
to run a make installworld from your source directory, which defaults to /usr/src. To run a make buildworld before the installworld, call:
```
ezjail-admin setup -b
```
For binary installations, ezjail uses the freebsd-update tool to keep the basejail up to date,:
```
ezjail-admin update -u
```
should do the trick.

# Image and crypto jails
Before the dawn of zfs, simple means to set limits on jails, like quotas, were hard to achieve. ezjail's answer were image jails, file backed "memory" disc images containing an ufs with the jail's content. When geom appeared with the very useful gbde and geli crypto layers for geom, encrypting image jails became possible. ezjail would handle creating and later attaching and detaching those images for you.

Now simple image jails are not as hot anymore and personally I would recommend using geli to encrypt the provider for your zpool to apply proper crypto to all your jails. Still, there may be valid use cases for image jail. Call:
```
ezjail-admin create -i -s 2G example.com 10.0.0.2
```
to create a two gigabyte md-image with an ufs file system and install the jail inside. To configure the jail without starting it, use the attach and detach subcommands of ezjail-admin config, like this:
```
ezjail-admin config -i attach example.com
cd /usr/jails/example.com
```
# … do your thing …
```
ezjail-admin config -i detach example.com
```
Should the file system need some love, e.g. after a spontanous reboot or system crash, call:
```
ezjail-admin config -i fsck example.com
```
to tidy up the mess–it ain't zfs, after all. By default ufs soft updates are enabled, so background fsck should occur for minor wrinkles, when an image jail starts.

To create encrypted image jails, use the -c switch and either pass bde or eli and follow the instructions on screen:
```
ezjail-admin create -c eli -i 16G example.com 10.0.0.3
```
Note, that ezjail creates image jails by filling them from /dev/zero or /dev/random, for performance reasons (reduce seeks with this file system inside a file system hack) and for security reasons (do not leak information about which blocks have been written for crypto jails), so creating huge image jails may take a while. Also note, that crypto jails would block the boot process (unless the passphrase is provided via a file or some fetch magic via stdin). So they are being marked as attachblocking and not started during boot time. You need to start them using ezjail-admin startcrypto.
