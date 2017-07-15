Originator Message version 2 (OGMv2)
====================================

|image0|
*Image Source*: `Martin Hundebøll, Jeppe Ledet-Pedersen, Network
Coding for Wireless Mesh
Networks <https://downloads.open-mesh.org/batman/papers/batman-adv_network_coding.pdf>`__

1. Definitions
--------------

-  Node - A mesh router which utilizes the B.A.T.M.A.N. protocol as
   specified in this document on at least one network interface.
-  originator - A node broadcasting its own OGMs that is therefore
   addressable within the mesh network routing layer. It is uniquely
   identifiable by its originator address. B.A.T.M.A.N.-Advanced uses
   the MAC address of its primary hard interface.
-  hard interface - Network interface utilized by B.A.T.M.A.N. for its
   own ethernet frames.
-  Neighbor: An originator within one hop distance.
-  Router: A neighbor which is a potential, loop-free next hop for
   forwarding data packets towards a specific originator.
-  link throughput: link throughput from one interface to a neighbor's
   interface (see :doc:`ELP <ELP>` for details).
-  best link throughput: The best of all link throughput values towards
   a neighbor (see :doc:`ELP <ELP>` for details).
-  path throughput: The incoming OGM's throughput combined with the best
   link throughput of the according neighbor.
-  Originator entry: Local data structure in the originator list (see
   'Conceptual Data Structures' for details).

Throughputs are specified in units of 100 kbit/s.

2. Conceptual Data Structures
-----------------------------

2.1. Originator List
~~~~~~~~~~~~~~~~~~~~

An originator list holds all addressable and to a certain degree
reachable originators within the mesh network. The Originator Address
and Selected Router fields of this list are of special interest for the
actual routing decisions upon incoming data packets.

-  Originator Address: The originator address of the node.
-  Originator's Sequence Number: The newest OGM Sequence Number that has
   been accepted from the given Originator.
-  Selected Router: A neighbor which is chosen as the next hop to
   forward data packets to for the according originator.

3. Protocol Procedure
---------------------

3.1 Broadcasting own Originator Message 2 (OGMv2)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Each node periodically (OGM interval) generates a single OGM which is
broadcasted on all hard interfaces. A jitter may be applied to avoid
collisions. The default interval in batman-adv is 1 second.

*The Originator Message 2 (OGMv2) Format:*

* Packet type: Initialize this field with the ELP packet type.
* Version: Set your internal compatibility version.
* TTL: Initialize with BATADV\_TTL
* Flags: not used
* Sequence number: On first broadcast set the sequence number to an
  arbitrary value and increment the field by one for each following
  OGMv2.
* Originator Address: Set this field to the primary MAC address of
  this B.A.T.M.A.N. node.
* TVLV length: Length of the TLVL data appended to the OGM
* Throughput: Throughput metric value in 100 kbit/s. Initialize with
  BATADV\_THROUGHPUT\_MAX\_VALUE
* TVLV data: Appended TVLV data for the originator. See :doc:`TVLV <TVLV>` for
  a detailed description.

::

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     | Packet Type   |    Version    |      TTL      |   Flags       |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                       Sequence Number                         |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                     Originator Address                        |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |  (cont'd) Originator Address  |  TVLV length                  |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                         Throughput                            |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                        TVLV data ...                          |

3.2. Receiving Originator Messages
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Upon receiving an OGM a node must perform the following checks before
the packet is further processed:

3.2.1. Preliminary Checks
^^^^^^^^^^^^^^^^^^^^^^^^^

The following checks are simple sanity checks which don't affect the
routing logic:

-  **Version Check:** If the OGM contains a version which is different
   to the own internal version the message must be silently dropped
   (thus, it must not be further processed).
-  **Source Check:** If the sender address of the OGM is an ethernet
   multicast (including broadcast) address the message must be silently
   dropped.
-  **Destination Check** If the destination address of the OGM is a
   unicast address which does not match the received interface MAC
   address, the message must be silently dropped.
-  **Own Message Check:** If the originator address of the OGM is our
   own the message must be silently dropped as this OGM originated from
   this node.

3.2.2. Metric Update
^^^^^^^^^^^^^^^^^^^^

The following steps check whether the Neighbor we received the OGMv2
from is a potential Router.

Each step is performed per potential outgoing interface where the OGMv2
may be rebroadcasted to allow
:doc:`Multi Link Optimizations <Multi-link-optimize>`. This a
**default** interface next to the configured hard interfaces, which is
used for locally generated traffic.

The following checks are performed before updating the metric:

* **Protection window check:** If the OGMv2s sequence number is older
  than BATADV\_OGM\_MAX\_AGE or newer than the
  BATADV\_EXPECTED\_SEQNO\_RANGE, and the protection window is active,
  the packet is silently dropped. If both conditions are met but the
  protection window is not active yet, the OGMv2 is allowed but the
  protection window gets activated.
* **Age check:** If the sequence number is strictly older than the
  last OGMv2, the packet is silently dropped. The only exception is when
  the protection window has just been activated, then the OGMv2 can
  pass.

If the initial checks above have passed, the internal stats are updated:

* the last seen timestamps of the router and the originator are
  updated
* the last sequence number and ttl values are adopted
* if the link throughput to the neighbor this OGMv2 was forwarded by
  is **lower** than the path throughput of the OGMv2, then this lower
  link throughput is adopted
* Forward penalties are applied:

* if the considered interface is the **default** interface, no
  penalty is applied
* if the incoming and considered outgoing interface is the same
  **half duplex** interface and the reported throughput is larger than 1
  MBit/s, the throughput is reduced by 50%
* Otherwise, a hop penalty is applied and the throughput is reduced
  by the according value (default 5.8% or 15/255). This is especially
  useful for "perfect" networks to create a decreasing metric over
  multiple hops.
* The throughput value with the penalties applied is stored for the
  router

3.2.3. Route Update
^^^^^^^^^^^^^^^^^^^

After that, we check the OGMv2 whether a router update should be done
and the OGMv2 should be rebroadcasted

* If the OGMv2 was received through a neighbor that is not (yet) a
  router, drop the OGMv2

The passing OGMv2 will be considered for a router update:

* If the OGMv2 has been received from the best router, no change is
  necessary
* If no router has been selected yet, the received router becomes the
  selected router immediately
* If the throughput from the received router is higher than the
  throughput via the selected router, the received router becomes the
  selected router
* Also, if the sequence number is by at least OGM\_MAX\_ORIG\_DIFF
  higher than the last received sequence number from the selected
  router, the received router becomes the selected router.

If the OGMv2 has been received by the (now) selected router, the OGM is
forwarded on the considered outgoing interface (except for the
**default** interface). However, the OGMv2 is not forwarded if another
OGMv2 has been forwarded with the same sequence number.

Furthermore, TVLV data is processed when this OGMv2 was newer than
previously received OGMv2s.

4. Re-broadcasting other nodes' OGMv2s
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When an OGMv2 is to be re-broadcasted some of the message fields must be
changed others must be left unchanged. All fields not mentioned in the
following section remain untouched:

* The TTL must be decremented by one. If the TTL becomes zero (after
  the decrementation) the packet must be dropped.
* The Path throughput for the considered outgoing interface is
  adopted

The OGMv2 is then rebroadcasted on the specific outgoing interface.

5. Values for Constants
-----------------------

BATADV\_THROUGHPUT\_MAX\_VALUE: 0xFFFFFFFF
BATADV\_TTL: 50
OGM\_MAX\_ORIG\_DIFF: 5
BATADV\_OGM\_MAX\_AGE: 64
BATADV\_EXPECTED\_SEQNO\_RANGE: 65536

.. |image0| image:: batman_ogm.svg

