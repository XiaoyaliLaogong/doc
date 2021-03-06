.. SPDX-License-Identifier: GPL-2.0

MeshLinux
=========

Meshlinux is outdated and really old. The project hasn't been updated in
a long time. It is unlikely that anyone will take up development on it
again. We recommend to use OpenWrt for X86 platform. The only purpose we
can think of today is software archaeology or the use of very old legacy
WiFi hardware.

Nothing is as outdated as old software. Meshlinux was created in the
early days of mesh networking at the Freifunk community in 2004. It is a
CD-Image that can be used to convert old PCs into wireless supernodes.
There is a old Meshlinux version based on Linux (Kernel 2.4.26)
available at (https://wiki.c-base.org/coredump/MeshLinux) This old
version is based on Slackware and not maintained anymore. However it
works nicely with old Atmel USB wireless devices (which is a problem
nowadays with recent kernels), Prism-based Chipsets and Orinoco. But you
won't find any modern mesh routing protocol implementations in it. If
you want to build a meshrouter with nothing better than a 486 CPU and
this kind of old wireless devices because you have them lying around and
want to do something useful with it, you may still use it and install a
recent version of Layer 3 B.A.T.M.A.N. - you can download a statically
compiled version from this server. You can not use Batman-advanced,
since this would require a recent Linux kernel. Some userland mesh
protocol daemons like OLSR might still work with it.

Sooner or later this information will be removed.
