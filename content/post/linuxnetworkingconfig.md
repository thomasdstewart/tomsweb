---
title: "Linux Networking Config"
summary: ""
authors: ["thomas"]
tags: ["linux", "networking"]
categories: []
date: 2022-03-30 11:30:00
---
Linux Networking Config is a complex beast these days. In fairness networking is complicated, and there has to be a way to configure a multiple of technologies: ethernet, wifi, ppp, vpn, mobile, bridge, bonding, vlan, tunnels. This is a short comparison of the options. Originally networking was configured during bootup in the shell scripts that form sysvinit. Over the 20+ years the below have popped up:

# Methods

## ifupdown derivatives
The main config for this is in /etc/network/interfaces and /etc/network/interfaces.d/, it's also called ENI. For Debian this is the original network setup after direct editing of boot up scripts. It's still the Debian installer default, the default for official Debian Cloud images and a dependency of cloud-init. You can archive more of less any static network configuration you want with some dynamic configurations being sort of doable. It's got quite a lot of hooks that do allow interesting integrations. However integration with other things like desktop tools does not exist, eg joining a random wifi network. Hence is quite common on servers but not for say a laptop. Also many implementations exist, the default Debian is ifupdown. I have no idea how well or not the other implementations work or not.

https://wiki.debian.org/NetworkConfiguration
https://www.debian.org/doc/manuals/debian-reference/ch05.en.html

### ifupdown
since woody
https://salsa.debian.org/debian/ifupdown
https://sources.debian.org/src/ifupdown/0.8.37/main.c/

### ifupdown (busybox)
since woody
https://busybox.net/
https://git.busybox.net/busybox/tree/networking/ifupdown.c

### ifupdown (netscript)
since sarge
https://packages.debian.org/sid/netscript-2.4
https://sources.debian.org/src/netscript-2.4/5.5.5/netscript/

### ifupdown2
since jessie
https://github.com/CumulusNetworks/ifupdown2/
https://github.com/CumulusNetworks/ifupdown2/blob/master/ifupdown2/ifupdown/main.py

### ifupdown-ng
since bookworm
https://github.com/ifupdown-ng/ifupdown-ng
https://github.com/ifupdown-ng/ifupdown-ng/blob/main/cmd/ifupdown.c

## network-scripts (redhat)
As I understand network-scripts were the first way networks were configured in Red Hat based distros after the directly editing rc scripts, as I understand the config files were sourced and then the scripts would then implement everything needed. These are stored under /etc/sysconfig/network-scripts. Almost all configurations were possible, however hooks to do "exotic" other things are harder. However these network-script have not been the only option on Red Hat based distros for quite sometime because Network-Manager was introduced in either RHEL 5 or 6, generally people recommended disabling it because people don't like changed. However for Network-manager has before the main way networking is configured, the say this worked was by implementing a sort of config compatibly layer that used the same config files, so you could run with both enabled and network would start some interfaces and network-manager would start other interfaces. However as time has gone on network-manager just works and with RHEL 8 its quite hard not to use it, and as I understand RHEL9 is just pain network manager.

since at least Red Hat Linux 6.0 (~circa 2000)

## ipconfig
This is quite a bizarre project and it is hard to see actual references to it. Also it's obviously hard to google as the name clash with the windows ipconfig tool. In essence it's a small tool that can do a simple dhcp request and set the response to it on an interface or just set an ip on an interfaces. It's usually called from the initrd and configured from the linux command line via the IP parameter. It's then can also unconfigure the interface after it's needed before chroot into the main system happens. So for example all this might be needed for ssh server in initrd, nfs root, or iscsi root. Also how it's called depends on the initrd used, usually initramfs or dracut.

since etch
https://git.kernel.org/cgit/libs/klibc/klibc.git
https://git.kernel.org/pub/scm/libs/klibc/klibc.git/tree/usr/kinit/ipconfig/main.c
https://git.kernel.org/pub/scm/libs/klibc/klibc.git/tree/usr/kinit/ipconfig/README.ipconfig

## NetworkManager
NetworkManager is an answer to some of the issues with the existing setup, mostly what seems to be around laptop type setups, where the network config changes all the time and needs gui type tools to do this. It's a daemon that handles this and implements changes. It's fair to say that initially people did seem to like it for servers, as people don't like changes. It's also works differently depending if you are on Debian or Red Hat based distributions, with the former using /etc/NetworkManager/system-connections/ and the latter using /etc/sysconfig/network-scripts. Initially it was not capable of implementing all the config that ifupdown is capable of, however as it's matured it's now able to implement almost all setups.

since etch
https://wiki.gnome.org/Projects/NetworkManager
https://gitlab.freedesktop.org/NetworkManager/NetworkManager
https://wiki.debian.org/NetworkManager

## systemd-networkd
systemd-networkd grew out of systemd, presumably with the intention of avoiding the mess of tools. It's idea for simple setups, eg just a static IP or DHCP ip. It then just works, without calling out to other tools and handling that mess, eg ifupdown calls dhclient, which has it's only config that might or might not do what is expected. However for anything complicated NetworkManager is probably a better fit.

since wheezy
https://systemd.io/
https://wiki.debian.org/SystemdNetworkd
https://wiki.archlinux.org/title/systemd-networkd

## Cloud-init
Cloud-init is a suite of tools that run on boot that can grab information from the hosting provider and do things with this information, eg inject ssh keys to a virtual machine. It can also configure the network too. It's got three methods to configure the network:

 * ENI - /etc/network/interfaces 
 * Networking Config Version 1 - DSL
 * Networking Config Version 2 - subset of netplan version 2

since wheezy
https://cloudinit.readthedocs.io/

## Netplan
At some point Canonical decided that the way network config was configured needed a rethink, and what seems to be along the lines of https://xkcd.com/927/ as best. It's a way to abstract the config and netplan to generate different config backends, currently either systemd-networkd or Network Manager. It uses it's own new DSL config of which there are two versions. I can sort of understand why this was done, but can't help but feel it adds a useless layer of config that should not be needed.

version 1 - DSL
version 2 - DSL
https://netplan.io/

# Remarks
