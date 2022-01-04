---
title: "Network Based LUKS Unlock"
summary: ""
authors: ["thomas"]
tags: ["linux", "luks", "debian"]
categories: []
date: 2022-01-04 16:57:00
---
Recently I wanted to see if I could make my public cloud based Linux infra more secure via LUKS (Linux Unified Key Setup) disk encryption. I realise that one must fully trust ones cloud provider, as they have acces to the hardware. However it would be nice to know that data is encrypted when stored on disk. This does not mitigate against a very bad cloud provider, as ultimtly if they are detemined enought they can get at the data. However implementing some sort of encryption does offer some protection against reading the data if disks are re-used and certently makes the barrier might higher for access.

# Selection
Internet searches quicky show up 4 options:
 1. Only encrypt a subset of data and have a dedicated disk thats encrypted and store the unlock key in /etc/crypttab.
 2. Implement mostly FDE (Full Disk Encryption) eg / but not /boot and enter the password to unlock at every boot in the console.
 3. Implement mostly FDE (Full Disk Encryption) eg / but not /boot and given it is in the cloud and on the Internet spawn a ssh in the initramfs and remotly unlock. (https://www.arminpech.de/2019/12/23/debian-unlock-luks-root-partition-remotely-by-ssh-using-dropbear/)
 4. Implement mostly FDE (Full Disk Encryption) eg / but not /boot and use some sort of network magic.

I quickly discared idea 1 and 2, as I don't want the unlock key stored on disk next to the encrypted disk and I don't want to enter a password at every boot.

So I continued to search for that magic part of option 3, and came across 3 solutions:
 1. clevis and tang as part of the The Network Bound Disc Encryption (NBDE) as part of the Policy-Based Decryption (PBD)
 2. kxd - Key exchange daemon (https://blitiri.com.ar/p/kxd/)
 3. some other one I can't find anymore, I'm sure I found another that looked workable, but can't find it anymore

Loosly both sounded fine, however the clevis and tang caught my attention because:
 1. Red Hat have sort of blessed it, eg it's a part of RHEL7, 8 and 9 Beta, so therefore will probably be maintained for the forseable. 
 2. The key is not entirly stored on the server side, which seemed fairly cool. Albeit if you are able to grab a disk image of the encrypted disk, you can probably get the key anyways.

# Clevis and Tang
Te project (https://github.com/latchset) have borrwed the namonglature of the Clevis fastner system (https://en.wikipedia.org/wiki/Clevis_fastener) and re-use the same set of termanology (clevis, tang, pin), in a cute but also confusing way. However loosly speaking, you have a tang server somwehere, and clevis uses a tang pin to connect to the tang server over a network to unlock the disk. The tick is that Clevis can run from initramfs with network support to unlock the root disk or main lvm volume with root and other disks on. This guide explains his very well: https://semanticlab.net/sysadmin/encryption/Network-bound-disk-encryption-in-ubuntu-20.04/

As an aside, there is another clevis pin that can use a TPM to unlock the disks too, which should allow promptless full disk encryption.

It's also fair to say that Red Hat also have some fair documentation about it, albeit fairly Red Hat Distro spesific:
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/configuring-automated-unlocking-of-encrypted-volumes-using-policy-based-decryption_security-hardening
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9-beta/html/security_hardening/configuring-automated-unlocking-of-encrypted-volumes-using-policy-based-decryption_security-hardening

# Issues
Before letting any of this loose on real infrasturcure I decided to play with it on a pair of VM's, one acting as the server and one acting as the client. With the intention that my real public cloud virtual machines would become clients, and the server would either run on another provider or at home.

Discoveries with Tang:
 * It's a network based daemon thats writtin in C (https://sources.debian.org/src/tang/8-3+deb11u1/src/tangd.c/)
 * The version packaged for Debian Stable (Bullseye) runs as root (https://sources.debian.org/src/tang/8-3+deb11u1/units/tangd%2540.service.in/ vs https://sources.debian.org/src/tang/11-1/units/tangd%2540.service.in/)
 * The version package for Debian Stable (Bullseye) writes to /var/db/tang, this is fixed in Debian Testing (Bookworm)
 * It's basically a http server
 * It does not have any ssl
 * It does not have any password protection

Discoveries with Clevis:

# Workarounds


# Bugs



