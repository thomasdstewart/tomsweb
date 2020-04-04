---
title: "Extra IDE"
summary: ""
authors: []
tags: ["linux"]
categories: []
date: 2009-11-04
---
ExraIDE is a patch to the Linux kernel to enable more that the standard 20 ide disk drives in a Linux system, as each IDE controller that is driven by the old style drivers be it PATA or SATA (ie not libata). Each ide channel takes up two drive slots and letters even if the physical card has only one sata disk per channel attached. This severely limits the total IDE disks in one system. This patch adds four extra major device numbers and the necessary bits to extend past ide[0-9] and hda[a-t] to ide[0-9a-d] and hda[a-zA]. The real fix is to improve the libata drivers to include support for my old broken controllers or upgrade to new sata controllers. In any case I have actually run with some form of this patch for the best part of 3 years. I was never really worth submitting to the main line, but I did [post](http://thread.gmane.org/gmane.linux.kernel/392817) it to the LKML.

[extraide.patch](extraide.patch)
