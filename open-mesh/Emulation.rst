.. SPDX-License-Identifier: GPL-2.0

Emulation Setup
===============

To give an answer to the often asked question "How to
test/evaluate/debug mesh network protocols?", this document explains a
virtual machine setup which can be used to test batman-adv in a
controlled environment. The idea is to use
`QEMU <http://www.qemu.org/>`__ or `KVM <http://www.linux-kvm.org>`__
(instead of pure simulation systems like
`NS-2 <http://www.isi.edu/nsnam/ns/)>`__ to run an unmodified Linux
system with the unmodified source code as it is used in real world
setups. Besides B.A.T.M.A.N., you could evaluate any routing protocol.

Architecture
------------

The test stack consists of the following components:

-  `OpenWrt <https://openwrt.org/>`__, any recent stable release for x86
   with minimal modifications (see below)
-  one (unmodified) `QEMU <http://www.qemu.org/>`__ instance per host
   (KVM or any other virtual machine)
-  one (modified, see below)
   `vde\_switch <http://wiki.virtualsquare.org/wiki/index.php/VDE_Basic_Networking>`__
   per instance. You need multiple instances to allow interconnection
   with wirefilter.
-  one (unmodified)
   `wirefilter <http://wiki.virtualsquare.org/wiki/index.php/VDE#wirefilter>`__
   per link between hosts.

OpenWrt
~~~~~~~

A standard OpenWrt can be downloaded and configured for X86. Some
packages (tcpdump, netcat, batman-adv...) should be selected and kernel
commandline parameters were modified to make sure that qemu boots up
properly:

::

    noapic acpi=off

Furthermore, to enjoy an automatic network device setup on boot you can
use the following script (save it as './files/etc/rc.local' in your
local OpenWrt build directory):

::

    #!/bin/sh

    # kill default openwrt network config
    NUM=$(ip link show eth0 | awk '/ether/ {print $2}'| sed -e 's/.*://' -e 's/[\n\ ].*//')
    ip addr add 192.168.$((0x$NUM)).2/24 dev  eth1
    ip link set up dev eth1
    ip link set down dev br-lan
    ip link del dev br-lan


    # setup batman, this step should be done before setting up the log server, otherwise it'll can't find log file
    ip link set up dev eth0
    echo "" > /proc/net/batman-adv/interfaces
    echo eth0 > /proc/net/batman-adv/interfaces
    echo 15 > /proc/net/batman-adv/log_level
    ip addr add 192.168.0.${NUM}/24 dev bat0
    ip link set up dev bat0

    # set up log server
    cat << EOF > /tmp/logserver.sh
    #!/bin/sh
    while [ 1 ]; do
        nc -l -p 2050 < /sys/kernel/debug/batman_adv/bat0/log
    done
    EOF
    chmod 755 /tmp/logserver.sh
    /tmp/logserver.sh&

The script above is far from perfect, it takes some minutes after the
rc.local file is finally started. Suggestions welcome.

prepare a working directory for your operations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

  $ mkdir qemu\_work
  $ cd qemu\_work

vde setup
~~~~~~~~~

Download vde2.3.1: https://sourceforge.net/projects/vde/files/vde2/
Download vde colour patch: attachment:vde2-2.3.1\_colour.patch

patch the "vde2-2.3.1\_colour.patch" for vde.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

  $ cd vde-2.3.1
  $ cp vde2-2.3.1\_colour.patch .
  $ patch -p1 < vde2-2.3.1\_colour.patch

compile and install vde
~~~~~~~~~~~~~~~~~~~~~~~

::

  $ cd vde-2.3.1
  $ ./configure
  $ make
  $ sudo make install

download and cp colourful.rc
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

  $ cp colourful.rc qemu\_work

vde\_switch
~~~~~~~~~~~

You can get detailed vde\_switch parameters documentation here:
http://www.linuxhowtos.org/manpages/1/vde\_switch.htm
The main advantage of
`vde\_switch <http://wiki.virtualsquare.org/wiki/index.php/VDE_Basic_Networking>`__
over
`uml\_switch <http://user-mode-linux.sourceforge.net/old/networking.html>`__
is that any clients can be attached to this virtual switch: QEMU, KVM,
UML, tap interfaces, virtual interconnections, and not just UML
instances.

If the vde\_switches were just connected with wirefilter "patch cables"
without modification, we would end up creating a broadcast domain and
switch loops which we don't want: The goal is to allow the packets to
travel only from one host to it's neighbor, not farther.

To accomplish this, the vde\_switch needs to be modified to have
"coloured" ports. The idea is:

-  each port has a "colour" (an integer number)
-  packets are only passed from ports to others with DIFFERENT colours.
-  packets are dropped on outgoing ports if it has the SAME colour as
   the incoming port.

In this concept, the host port can have colour 1 while the
interconnection ports have colour 0. This way, packets can only travel
from the host to (all of) the interconnection ports, or from one
interconnection port to the host port. However packets can not travel
between the the interconnection ports, thus only allowing "one hop"
connections and avoiding switch loops and shared broadcast domains. The
concept is illustrated below:

|image0|

You can find the patch against vde2-2.3.1 (current latest stable
version) to add this colour patch here:

-  attachment:vde2-2.3.1\_colour.patch

wirefilter
~~~~~~~~~~

Wirefilter manpage:
http://manpages.ubuntu.com/manpages/trusty/man1/wirefilter.1.html
wirefilter is a tool where you can simulate various link defects and
limits:

-  packet loss
-  burst loss
-  delay
-  duplicates
-  bandwidth
-  noise (damage to packets)
-  mtu
-  ...

However as the links are only set up bidirectional, interferences can
unfortunately not be simulated with this system.

For advanced testing it might be necessary to apply the aforementioned
link defects to some packets only whereas other packets are able to
traverse the emulated environment unharmed. Once you applied the
'ethertype' patch you can specify an ethertype which wirefilter will
simply forward. To apply a packet loss of 50% to all packets except
batman-adv packets, run:

::

    wirefilter --ether 0x4305 -l 50

This patch also allows to filter batman-adv packet types. To apply a
packet loss of 50% to all packets except batman-adv ICMP packets, run:

::

    wirefilter --ether 0x4305:02 -l 50

You can specify up to 10 packet types (separated by colon). The patch
against vde2-2.3.1 (current latest stable version) can be found here:

-  attachment:vde2-2.3.1-wirefilter-ethertype.patch

copy openwrt-x86-generic-combined-ext4.img to your qemu\_work directory.

Scripts
-------

The following script is used to start up all the qemus. It is a good
idea to start the script inside a screen to have the QEMU instances in
screen windows (which can be switch with ctrl+a n, ctrl+a p). Make sure
that you have the correct sudo priveleges or alternatively run this
script as root.

The script does:

-  kill old instances
-  start up vde\_switch instances for each host
-  start up QEMU hosts (one Ethernet tap device is created per instance
   to allow logging etc)
-  install the links between the hosts. The resulting topology will be
   similar to this:
   |image1|

::

    #!/bin/sh
    QEMU=qemu-system-x86
    VDESWITCH=vde_switch
    IMAGE=openwrt-x86-generic-combined-ext4.img

    if [ "$TERM" != "screen" ];
    then
    echo "Must be run inside a screen session" 1>&2
    exit 1
    fi

    # you can set this if you are running as root and don't need sudo:
    # SUDO=
    SUDO=sudo

    ${SUDO} killall -q qemu
    killall -q wirefilter
    killall -q vde_switch

    for i in $(seq 1 9);
    do
        ${VDESWITCH} \
            -d --hub --sock num${i}.ctl -f colourful.rc
        ${SUDO} ip tuntap add tapwrt${i} mode tap
        ${SUDO} ip addr add 192.168.${i}.1/24 dev tapwrt${i}
        ${SUDO} ip link set up dev tapwrt${i}
    done

    for i in $(seq 1 9);
    do
        cp ${IMAGE} num${i}.image
        screen -dmS num${i} ${QEMU} \
            -no-acpi -m 32M \
            -net vde,sock=num${i}.ctl,port=1 -net nic,macaddr=fe:fe:00:00:$(printf %02x $i):01 \
            -net nic -net tap,ifname=tapwrt${i},script=no,downscript=no \
            -nographic num${i}.image
        sleep 3
    done

    wirefilter --daemon -v num1.ctl:num2.ctl
    wirefilter --daemon -v num2.ctl:num3.ctl
    wirefilter --daemon -v num3.ctl:num4.ctl
    wirefilter --daemon -v num4.ctl:num5.ctl
    wirefilter --daemon -v num5.ctl:num6.ctl
    wirefilter --daemon -v num6.ctl:num7.ctl
    wirefilter --daemon -v num7.ctl:num8.ctl
    wirefilter --daemon -v num8.ctl:num9.ctl

    wirefilter --daemon -v num1.ctl:num3.ctl -l 60
    wirefilter --daemon -v num3.ctl:num5.ctl -l 60
    wirefilter --daemon -v num5.ctl:num7.ctl -l 60
    wirefilter --daemon -v num7.ctl:num9.ctl -l 60
    wirefilter --daemon -v num2.ctl:num4.ctl -l 60
    wirefilter --daemon -v num4.ctl:num6.ctl -l 60
    wirefilter --daemon -v num6.ctl:num8.ctl -l 60

About IMAGE variable:

create this script in the "qemu\_work" directory that you have created
before.
when you use openwrt chaos\_calmer version, you should use
openwrt-x86-generic-combined-ext4.img.

The script is using the switch configuration file "colourful.rc". It
creates the ports (create more if your topology demands this) and sets
the host port (port no. 1) colour to 1. Put the following text in this
file:

About screen command
~~~~~~~~~~~~~~~~~~~~

::

    #show all screen instances
    $screen -ls

    #you can login a specific openwrt system with following command
    $screen -r num1          #(num1~num9)

::

    port/setcolourful 1
    port/create 1
    port/create 2
    port/create 3
    port/create 4
    port/create 5
    port/setcolour 1 1

To collect the batman logs from the individual hosts, you might want to
use this script after all nodes have completed booting and started
batman:

::

    #!/bin/sh
    for i in $(seq 1 9)
    do
        nc 192.168.${i}.2 2050 > num$i.log &
    done

.. |image0| image:: vde_switch.png
.. |image1| image:: mesh.gif

Resources
---------

* :download:`vde2-2.2.3_colour.patch`
* :download:`vde2-2.3.1-wirefilter-ethertype.patch`
* :download:`vde2-2.3.1_colour.patch`
* :download:`vde2-2.3.2-wirefilter-ethertype.patch`
* :download:`vde2-2.3.2_colour.patch`
* :download:`vde2-2.3.2_frame_size.patch`
