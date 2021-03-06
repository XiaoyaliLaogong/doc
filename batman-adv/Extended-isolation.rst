.. SPDX-License-Identifier: GPL-2.0

Extended-isolation
==================

This feature is an extension of the existing :doc:`AP-isolation <Ap-isolation>` that aims
to improve the old mechanism while keeping a backward compatible
behaviour.

When the extended-isolation is enabled batman-adv drops all the
wirelsss-to-wireless traffic and at the same time marks clients as
*isolated* based on the user configuration.
This way batman-adv provides a mechanism to achieve the following
goals:

#. isolate not only wireless clients but also all those hosts matching a
   particular user criteria
#. have generic firewall rules at the receiving nodes affecting traffic
   generated by clients marked as isolated

How to recognise isolated clients
---------------------------------

In the classic AP-Isolation mechanism clients are marked as *WIFI*
when packets originated by them reach the batman-adv interface after
having being received on a wireless interface.
With the Extended-isolation instead the idea is to classify as
*isolated* every client which packets are (fw)marked with a given
value. This increases the flexibility of the whole mechanism since the
user can setup any creative firewall rule to make the host mark
packets only when matching particular conditions.

A simple use case might be to mark all the packets coming in through a
given interface by using *tc*:

::

    # tc qdisc add dev eth1 handle ffff: ingress
    # tc filter add dev eth1 parent ffff: protocol all pref 10 u32 match u32 0 0 flowid 1:1 action skbedit mark 0x6

With these commands, all the the packets coming in through *eth1* will
be marked with the fwmark 0x6.

The next step it is to configure batman-adv properly, meaning that the
user must tell the kernel module which is the value to match in the
*skb->mark* field when detecting isolated clients.

::

    # echo 6/0xffffffff >/sys/devices/virtual/net/bat0/mesh/isolation_mark
    or
    # batctl isola 0x06/0xFFFFFFFF # VALUE/MASK

As reported in the example above, the isolation mark needs to be
configured in batman-adv in the form *VALUE/MASK* in order to keep the
mechanism as flexible as possible.
The fwmark value attached to a given packet can be (and usually is)
used by several components in the kernel and therefore, to avoid
colliding and breaking those other mechanisms, batman-adv should work
only on a limited number of bits within this field. The MASK part is a
(hexadecimal) bitmask representing which bits batman-adv is allowed to
work with, while the VALUE represent the real value that has to be
used. Due to the restriction given by MASK, the VALUE is altered in
order to avoid touching any bit that is not selected by the former.

How to transfer the mark from node to node
------------------------------------------

The previous sections explained how to detect clients that have to be
marked as *isolated*, but how is this information passed to other
nodes in the network?
Clients recognised as *isolated* are marked with a special TT flag,
namely TT\_CLIENT\_ISOLA, which is then spread all over the network
with the rest of the :doc:`TT information <Client-announcement>` so that
other nodes are aware of which global client has to be considered as
isolated.

Dropping logic
--------------

Now that all the nodes in the network have the needed information there
are two possible scenarios:

#. a unicast packet has to travel over the mesh from an isolated client
   to another isolated client => the packet is dropped at the source
   node
#. a broadcast packet is sent by an isolated client => every receiving
   node marks the locally generated copy of the packet with the
   configured isolation mark. This way any firewall rule on the
   receiving endpoint can play the same game like if the packets were
   locally generated.
   The marking upon reception mechanism is possible thanks to the
   TT\_CLIENT\_ISOLA flag that has been previously spread in the network
   (the original mark is not involved at all in this operation).

Note that each node could have its own isolation mark and there is no
need for them to be the same.

How to drop broadcast traffic
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Following the example reported in the previous sections, a set of tc
commands can help to drop all the broadcast packets having been marked
with a given fwmark:

::

    # tc qdisc add dev eth1 root handle 1: prio
    # tc qdisc add dev eth1 parent 1:3 handle 30: netem loss 100%25
    # tc filter add dev eth1 parent 1:0 protocol all prio 1 handle 6 fw classid 1:3

With this instructions two task are accomplished:

#. a netem queue is created telling to drop 100%25 of the packets
   entering it
#. a filter rule is added in order to redirect all the traffic matching
   the specified mark to the netem queue.

Note that *tc* is used instead of iptables because the latter would
only be able to work on frames carrying IPv4/6 packets, while in this
case the target is the entire traffic.
However, any kind of rule interacting with the fwmark can be used to
implement any wanted behaviour.

Example
-------

|image0|

In the picture above traffic is allowed only between the wired and the
non-isolated wireless interface (both the interfaces must be
"isolated" to block the traffic).
Wireless to wireless traffic is blocked due to the already existing
:doc:`AP-isolation <Ap-isolation>` mechanism, while between the two wired "isolated"
interfaces there is no communication because of the new Extended
Isolation feature described in this page.

.. |image0| image:: ext-isola.png

