.. SPDX-License-Identifier: GPL-2.0

========================
Originator Message (OGM)
========================

This page is work-in-progress and will state the BATMAN V algorithm
later.

1. Introduction
===============

The B.A.T.M.A.N. protocol originally only used a single message type
(called OGM) to determine the link qualities to the direct neighbors and
spreading these link quality information through the whole mesh. This
procedure is summarized on the :doc:`BATMAN concept page </open-mesh/BATMANConcept>` and explained in details in `the RFC
draft <https://tools.ietf.org/html/draft-wunderlich-openmesh-manet-routing-00>`__
published in 2008.

With the new concept of a separate :doc:`ELP <ELP>` the tasks performed by
OGMs becomes way simpler: It only determines and choses the best router
towards another originator without having any knowledge about possibly
multiple interfaces, while any link specific work is done by ELP.

This bears the following advantages from the OGM point of view:

-  Reduced overhead, as OGMs can then be send with a slower interval.
   The BATMAN routing algorithm still has a squared amount of overhead
   in worst case scenarios, therefore the the slower intervals are very
   desireable.
-  The BATMAN core routing protocol becomes less entangled with other
   mechanisms and features, making it easier understand and to perform
   theoretical analysis on.
-  Easier to optimize the OGMs convergence speed.

2. Definitions
==============

-  Node - A mesh router which utilizes the B.A.T.M.A.N. protocol as
   specified in this document on at least one network interface.
-  originator - A node broadcasting its own OGMs (see
   :ref:`section 4.1 <batman-adv-OGM-41-Broadcasting-own-Originator-Message-OGM>` for
   details) that is therefore addressable within the mesh network
   routing layer. It is uniquely identifiable by its originator address.
   :doc:`B.A.T.M.A.N.-Advanced </index>` uses the MAC address of its primary hard
   interface.
-  hard interface - Network interface utilized by B.A.T.M.A.N. for its
   own ethernet frames.
-  sliding window - Sequence numbers are recorded in dedicated sliding
   windows until they are considered out-of range. Thus, such a sliding
   window always contains the set of recently received sequence numbers.
   The amount of sequence numbers recorded in the sliding window is used
   as a metric for the quality of detected links and paths.
-  duplicate - A received OGM message from a neighbor containing an
   already received sequence number.
-  out of order - A received OGM message from a neighbor containing a
   sequence number that is older than the newest sequence number ever
   received from this neighbor.
-  Neighbor: An originator within one hop distance.
-  Router: A neighbor which is a potential, loop-free next hop for
   forwarding data packets towards a specific originator.
-  link TQ: link TQ from one interface to a neighbor's interface (see
   ELP for details).
-  best link TQ: The best of all link TQ values towards a neighbor (see
   ELP for details).
-  path TQ: The incoming OGM's TQ multiplied with the best link TQ of
   the according neighbor.
-  Originator entry: Local data structure in the originator list (see
   'Conceptual Data Structures' for details).
-  Router entry: Local data structure in the router list (see
   'Conceptual Data Structures' for details).

The TQ is a number between zero (worst) and one (best). A TQ in a packet
is linearly mapped to one byte, 0x00 representing a zero TQ, 0xFF a TQ
of one.

3. Conceptual Data Structures
=============================

3.1. Originator List
--------------------

An originator list holds all addressable and to a certain degree
reachable originators within the mesh network. The Originator Address
and Selected Router fields of this list are of special interest for the
actual routing decisions upon incoming data packets.

-  Originator Address: The originator address of the node.
-  Originator's Sequence Number: The newest OGM Sequence Number that has
   been accepted from the given Originator.
-  Router List: A list of potential routers towards an originator
-  Selected Router: A router from the Router List which is chosen as the
   next hop to forward data packets to for the according originator.

3.2 Router List
---------------

A router list holds all potential routers, routers that might be
switched to at any time without creating a routing loop. An entry
buffers the newest, valid OGM from the according router and according
originator of the OGM to be able to rebroadcast an OGM later when
switching to another potential router.

-  Router Originator Address: Originator address of the router an OGM
   was received from.
-  Rebroadcasted: This flag indicates whether the current Router Entry
   settings have been rebroadcasted yet.
-  Received OGM, the following properties are of special interest for
   the routing decisions:

   -  Path TQ: The path TQ towards the originator of the OGM multiplied
      with the best link TQ at the time of reception.
   -  Router's Sequence Number: The sequence number of the last accepted
      OGM received via the according router.
   -  TTL: The TTL of the last accepted OGM received via the according
      router.

-  As well as all other properties of the OGM like:

   -  Interval, Flags, Gateway Flags, TT Num Changes, TT VN, TT CRC and
      a possible TT change entry (see :doc:`Client-announcement <Client-announcement>` for TT
      related field descriptions)

4. Protocol Procedure
=====================

.. _batman-adv-ogm-41-broadcasting-own-originator-message-ogm:

4.1 Broadcasting own Originator Message (OGM)
---------------------------------------------

Each node periodically (OGM interval) generates a single OGM which is
broadcasted on all hard interfaces. A jitter may be applied to avoid
collisions.

*The Originator Message (OGM) Format:*

-  Packet type: Initialize this field with the OGM packet type.
-  Version: Set your internal compatibility version.
-  Interval: Set to the current OGM interval of this originator in milli
   seconds.
-  Sequence number: On first broadcast set the sequence number to an
   arbitrary value and increment the field by one for each following
   broadcast.
-  Originator Address: Set this field to the primary MAC address of this
   B.A.T.M.A.N. node.
-  Flags: Indicates certain attributes of this originator. So far 0x01
   is reserved for VIS\_SERVER (see :doc:`VisAdv <VisAdv>`)
-  Gateway Flags:
-  TQ: Is initially set to TQ\_MAX by the originator.
-  TT Num Changes: see :doc:`Client-announcement <Client-announcement>`
-  TT VN: see :doc:`Client-announcement <Client-announcement>`
-  TT CRC: see :doc:`Client-announcement <Client-announcement>`

::

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |  Packet Type  |    Version    |     TTL       |   Alignment   |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                          Interval                             |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                       Sequence Number                         |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                     Originator Address                        |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |      Originator Address       |     Flags     | Gateway Flags |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |      TQ       |TT Num Changes |     TT VN     |    TT CRC     |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

As well as a possible TT change entry. See :doc:`Client-announcement <Client-announcement>` for
details.

4.2. Receiving Originator Messages
----------------------------------

Upon receiving an OGM a node must perform the following checks before
the packet is further processed:

4.2.1. Preliminary Checks
~~~~~~~~~~~~~~~~~~~~~~~~~

-  If the OGM contains a version which is different to the own internal
   version the message must be silently dropped (thus, it must not be
   further processed).
-  If the sender address of the OGM is an ethernet multicast (including
   broadcast) address the message must be silently dropped.
-  If the destination address of the OGM is a unicast address the
   message must be silently dropped.
-  If the originator address of the OGM is our own the message must be
   silently dropped as this OGM originated from this node.

4.2.2. Potential Router Checks
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following steps check whether the Neighbor we received the OGM from
is a potential Router, meaning that we could switch to this Neighbor
without creating a routing loop. If this is not the case we are going to
drop and ignore this OGM. Otherwise we will further call this Neighbor a
potential Router or just Router and will pass on to the Router Ranking.

-  If an originator entry matching the originator address of the OGM and
   a Selected Router exist:

   -  If the OGM's Sequence Number is smaller than the Selected Router's
      Sequence Number then the message must be silently dropped. This
      step is needed to ensure loop-freeness, we may only select newer
      or in certain circumstances equal sequence numbers.
   -  If for the according originator entry's router list a router entry
      matching the neighbor we received the OGM from exists and this
      entry has a sequence number higher than the one in the OGM then
      the message must be silently dropped. Due to the previous check
      this step is not needed to ensure loop-freeness. Instead it
      ensures that we are not "updating" a router entry (which might not
      be the Selected Router at the moment) with older information.

If the OGM has not been dropped after these preliminary checks, the OGM
will be modified in the following way to obtain the path TQ of the
received OGM:

-  The OGM's TQ field needs to be multiplied with the best link TQ
   towards the according neighbor. This new TQ value is further
   referenced as path TQ.

A final check then needs to be applied:

-  If an originator entry matching the originator address of the OGM and
   a Selected Router exist:

   -  If the OGM's Sequence Number is equal to the Selected Router's
      Sequence Number and the OGM's path TQ is lower than the Selected
      Router's path TQ then the message must be silently dropped. This
      step is needed to ensure loop-freeness, an OGM of the same
      sequence number and a lower path TQ might have been rebroadcasted
      from us before and might have made any next hop along the selected
      path to have chosen us a a next hop again, possibly creating a
      routing loop. And actually we are just interested for the best
      path TQ for now anyway.
   -  If for the according originator entry's router list a router entry
      matching the neighbor we received the OGM from exists and this
      entry has a sequence number equal to the one in the OGM and the
      OGM's path TQ is lower than or equal to the router entry's path TQ
      then the message must be silently dropped. Due to the previous
      check this step is not needed to ensure loop-freeness. Instead it
      ensures that we are only updating a router entry with the same
      sequence number to a better path TQ (which might have arrived over
      a "longer", more delayed path). This is also needed to ensure in
      case of very low packet loss over best paths to get the best, true
      path TQ values within the OGM flood of one sequence number.

5. Router Ranking
=================

For each OGM having passed the previous checks the according neighbor is
a potential, loop-free router. The Router Ranking checks whether just an
according Router or even completely new Originator entry needs to be
created or an already existing Router entry matching the Router we
received this OGM from just updated. Or whether also the currently
Selected Router needs to be switched. Furthermore step 5.2. will force
relinquishing the so far Selected Router if its information became too
old because of this OGM received via a Router other than the Selected
Router.

If this OGM just results in updating a Router in the Router list which
is not and not going to be the currently Selected Router, then no
rebroadcasting of this OGM will take place in step 5.3. for now.

Finally, any Neighbors which are not loop-safe Routers anymore after a
possibly newly Selected Router will be removed from the Router list in
step 5.4.

For the Router Ranking the following actions must be performed:

5.1. Creating or Updating Originator and Router Entries
-------------------------------------------------------

In this step we are updating the according entries in the Router list.
Step 4.2.2 ensured that the OGM which was not dropped yet is actually
containing either newer (higher sequence number) information - which is
a loop-safe, possible choice because that Router or any next hop on that
path did not and will not be allowed to switch back to a lower sequence
number again (like the lower sequence number we would have). Or
information of the same originator's OGM flooding round, the same
sequence number, as the currently Selected Router but with that OGM
having travelled along a better path (better due to a higher path TQ of
this OGM - and note that an OGM having travelled along such a better
path can never have travelled over us before, as then the path TQ would
have to be worse and not better in such a case as with each hop the path
TQ gets at least 1/255 worse due to the Hop Penalty, see section 7.1).
The OGM with the properties just stated might also have been received
from a Neighbor which we do not have a Router entry - or even an
Originator entry yet which will be created in that case first.

More precisely, the following steps need to be undertaken in the
updating and creating process:

-  If no originator entry matching the originator address of the OGM
   exists:

   -  Create a new originator entry with the originator address and
      originator's sequence number set to ones from the OGM.

-  If no router entry matching the OGM's originator and the neighbor the
   OGM was received from exists:

   -  Create a new router entry with the Router Address set to the
      address of the Router we received the OGM from. Buffer the
      complete OGM in this entry.
   -  Unset this new router entry's Rebroadcasted flag.

-  Otherwise:

   -  Delete the old buffered OGM and buffer this newly received OGM
      instead.
   -  Unset the Rebroadcasted flag in this matching router entry.

5.2. Purging Outdated Router Entries
------------------------------------

It might happen that for instance from a certain Neighbor we would
receive an OGM of perfect quality first and will chose that Router.
However after that the path over that Selected Router could suddenly be
jammed, leading to no more updates from that Router, resulting in a
stale entry. Newer and newer (higher sequence number) OGMs might arrive
over other potential Routers, but would never be chosen because of a
path TQ never being better than perfect, highest path TQ of the
currently Selected Router. Therefore we need to at some point consider
this Selected Router as outdated and switch to one of the alternative,
loop-free Routers in our list which provide more up-to-date information.
This is not being done time-based but based on the sequence number, a
Selected Router may only be chosen if its OGM has not been older than
OGM\_SEQ\_RANGE sequence numbers.

Note that a lower OGM\_SEQ\_RANGE favours chosing Routers with the
most-up-to date information: This especially penalizes asymmetric links
and paths - although we do not receive that many OGMs from such a Router
with such an asymmetric path (showing a not that good receive quality),
it might still be the best choice for transmitting our own data packets
though. And could lead to fast route flapping also in symmetric
topologies when OGMs in general have a low probability of arrival.
However having a too large OGM\_SEQ\_RANGE might favour too old,
outdated information too much, as described with the example before.

More precisely we have to:

-  If the OGM's Sequence Number is newer than the Originator's Sequence
   Number:

   -  The new Originator's Sequence Number must be set to the Sequence
      Number contained in the received OGM.
   -  for all Routers of the OGM's originator: if (Originator's Sequence
      Number - Router's Sequence Number) > OGM\_SEQ\_RANGE, purge the
      router from the OGM's originator's Router List.

Note that neither applying this outdated Router purging harms
loop-freeness as we would Select a new Router with a higher sequence
number in section 5.3. and again, the Router that would be selected next
or any next hop behind it would not have selected us or will not select
us due to them not being allowed to switch back to a lower sequence
number again. Nor is this purging of outdated Routers needed to ensure
loop-freeness. It is just an optimization for certain scenarios as
described previously.

Also note, that this step can result in rebroadcasting an OGM in step
5.3. which is not the one we have actually received and are currently
processing - which is intended: This incoming OGM might be the cause of
purging outdated entries, however there might be still other loop-free
Routers in the Router list which have a higher path TQ and are therefore
more desirable to chose as the new Selected Router than the Router we
received this OGM from.

5.3. Switching to (or Keeping) best Router
------------------------------------------

This step ensures a good Router selection to the best knowledge of a
node. As the Router list only keeps potential, loop-free nodes (due to
steps 4.2.2 and 5.4) which are further not too old (due to step 5.2) we
can now freely choose any node from this list. If in this round we got
an OGM of a Router which we did not and will not chose as the Selected
Router (due to a lower path TQ, although it would be feasible to chose
it due to a newer sequence number of this OGM), than we just updated
this Routers values, without selecting it. Instead the next steps will
chose the same old Router (which is not the one we received the current
OGM from) again - but will avoid rebroadcasting the OGM of the old,
though still best old and newly Selected Router, due to the
Rebroadcasted flag.

Specifically, we must undertake the following actions:

-  Set the Selected Router to the Router with the highest path TQ.
-  If the Selected Router's Rebroadcasted flag is not set:

   -  Rebroadcast the OGM of this Selected Router.
   -  Set the Selected Router's Rebroadcasted flag.

5.4. Purging non-potential Routers
----------------------------------

When rebroadcasting a new OGM certain other Routers do not guarantee
loop-freeness anymore. We can still chose the Routers from our list that
either have broadcasted a higher sequence number than the one we might
have just rebroadcasted, they or any next hop behind them are not
allowed to switch their route to a lower sequence number (like the one
we might just have rebroadcasted) again. Or we could chose any router in
the list with the same sequence number and a higher path TQ than the one
of the Selected Router (though this will not be the case, because step
5.3. ensured that we are already chosing the Router with the highest
path TQ from our Router list). In all other cases we do not consider
these Neighbors as potential Routers anymore, they or any next hop
behind them might start chosing us as their router without us noticing.
Therefore we remove them from the list to ensure a safe Router list:

If an OGM was rebroadcasted in the previous step:

-  Purge all routers with a sequence number smaller than the Selected
   Router's Sequence Number.
-  Purge all routers with a sequence number equal to the Selected
   Router's Sequence Number and a path TQ smaller than the Selected
   Router's path TQ.

Note: If no OGM was rebroadcasted in the previous section then no
purging will be done in this section anyways. However the "If" shall
emphasize, that not the switching of the Selected Router makes the
router list clean-up in this section mandatory to ensure loop-freeness,
but the rebroadcasting of an OGM does.

6. Re-broadcasting other nodes' OGMs
------------------------------------

When an OGM is to be re-broadcasted some of the message fields must be
changed others must be left unchanged. All fields not mentioned in the
following section remain untouched:

-  The TTL must be decremented by one. If the TTL becomes zero (after
   the decrementation) the packet must be dropped.
-  The hop penalty must be applied on the OGM's TQ field. See 'Penalties
   - Hop Penalty' for further details. If the OGM's TQ becomes zero
   (after hop penalty) the packet must be dropped.

7. Penalties
============

7.1 Hop Penalty
---------------

In certain network setups the link quality between neighbors is very
similar whereas the number of hops is not. In these scenarios it is
desirable to chose the shortest path to reduce latency and to safe
bandwidth (especially on wireless mediums). The hop penalty is a value
greater than zero and smaller or equal to one. It is a fixed value but
may be changed during runtime. The hop penalty is applied on an outgoing
OGM in the followig way:

-  Outgoing OGM's TQ = path TQ \* (1 - hop penalty)

The result always needs to be rounded down to ensure that an outgoing
OGM's TQ is always smaller than the incoming OGM's TQ.

8. Proposed Values for Constants
================================

OGM\_SEQ\_RANGE: 5

TQ\_MAX: 0xFF

Appendix
========

Questions
---------

-  Where to add the description of the data packet forwarding + bonding
   mode description? Which object should the bonding interface stuff be
   added to?
-  Should the further optimization 'Resend OGMs with flags etc. of
   newest OGM' already be part of the standard?

Notes
-----

-  Section 'Receiving other nodes' OGMs' ensures the loop-freeness - any
   OGM having past that part and is accepted and loop-safe and can
   potentially be used as a (new) router
-  'Router Ranking' purges routers that are outdated in terms of
   sequence number and selects the routers with the highest path TQ. It
   further updates the status of the according router the OGm was
   received from.
-  With the simple ELP link quality handling the EIGRP/BABEL feasibility
   is not necessary as any router we could switch to due to EIGRP
   feasibility but not due to DSDV feasibility is actually a *worse*
   choice as that router would have a lower path TQ.
-  The ELP link quality information could possibly be made more use of.
   For now, it is always applied very early when receiving the OGM and
   never considered anymore. It could allow us to (a) switch to another
   router when the link quality of our currently selected router got
   worse without needing a higher sequence number of the other router
   (with the help of EIGRP feasibility). Or (b) allow us to avoid
   switching to a new, other router upon receiving an OGM from a router
   other than our selected router because the link quality towards our
   currently selected router (and therefore its path TQ) increased a lot
   since we last received the OGM of our currently selected router. So
   there is potential to optimize things within the *same* sequence
   number, but that'd make things more complex and error prune of
   course. The way it is stated here for BATMAN V at the moment should
   be rather straight-forward and clear and make it rather unlikely that
   we'd miss a case where a routing loop could occure (both conception
   and implementation wise).
-  OGM\_SEQ\_DIFF: The larger, the more we are (a) relying on / sticking
   to maybe outdated path TQ information and (b) giving asymmetric links
   a chance to being chosen as a route, © possibly reducing route
   flapping towards asymmetric links and (d) possibly reducing route
   flapping in low path TQ topologies where only every X OGMs arrive
   anyway.
-  (If a node does not rebroadcast OGMs, it could safely switch the
   route to any neighbor at any time; if it only rebroadcasts some, it
   might switch to some other routers more often - routes are getting
   EIGRP feasible more often. Maybe some potential to optimize things in
   the 'Router Ranking here later?)

Changes between OGMs in BATMAN IV and BATMAN V
==============================================

Renaming
--------

-  Global TQ renamed to Path TQ
-  Local TQ renamed to Link TQ

Removal of Global TQ Window and gl. TQ Averaging
------------------------------------------------

Only remember last Path TQ and its sequence number as well as the
highest
sequence number of an originator node received. The window size is
substituted by a
SEQ\_DIFF\_MAX (default: 5). A neighbor node is being purged from the
router list
if orig\_node\ [STRIKEOUT:last\_seqno] router->last\_seqno >
SEQ\_DIFF\_MAX.

Removed previous sender field
-----------------------------

Bonding Mode using NDP's link qualities
---------------------------------------

Removal of secondary interface originators
------------------------------------------

Instead when receiving an OGM, always the best link quality measured
by NDP
will be substracted from the OGM. This shall make multiple interfaces
transparent from the OGM algorithm.

Removal of PRIMARIES\_FIRST\_HOP + DIRECTLINK flag
--------------------------------------------------

PRIMARIES\_FIRST\_HOP flag is no more needed as there will be just one
primary originator a.k.
originator.

Removal of bidirectional link check
-----------------------------------

Asymetric Penalty should be better and enough.

(Re)Introduce strict OGM forwarding policy
------------------------------------------

To avoid routing loops. MGO/batping mechanism will compensate for
convergence speed performance.

Optimized Route Switching in case of outdated currently selected router
-----------------------------------------------------------------------

Before in BATMAN IV if a node was receiving an OGM with a sequence
number that caused the
currently selected router to be moved outside of the global TQ window
(e.g. receiving an
OGM with an originator's sequence number + 5) the route were switched
to the router this
OGM just came from. However although the sequence number is very new
and ensures loop-freeness,
the global TQ via this hop might be very low and therefore this router
being a bad choice.
As the now newly selected, possibly bad router has the highest
sequence number, it is
more "difficult" than necessary for another, better neighbor to become
the new router.

In BATMAN V, when the sequence number of the currently selected router
becomes too low,
a node may switch to a different router, a neighbor other then the one
we just received the OGM from:
It will switch to the router with the highest path TQ which is still
in the OGM\_SEQ\_DIFF\_LIMIT
and rebroadcast its buffered OGM instead of the just received OGM. The
just received OGM will
still be buffered (router-addr + seqno + path-TQ) though.

Strict hop-penalty
------------------

In BATMAN V the hop penalty always decreases the OGM's path TQ (at
least by 1/255: minimum selectable
hop penalty: 1, path TQ always rounded downwards)

Routing loops could potentially occure if a link has 0% packet loss and:

-  Either if the hop penalty is set to 0.
-  (Or if the received OGMs path TQ is very low (and hop penalty would
   not change the path TQ
   due to rounding issues?)

Question: Or is the TTL check in the Router Ranking actually enough?
Anyways, I guess having always
monotonically decreasing path TQ values of an OGM upon rebroadcasts is
probably also easier to prove
to be loop-free. And shouldn't harm anything - therefore I'd have a
better feeling with that change.

Increase NDP window size to 128 packets
---------------------------------------

With removing the Path TQ averaging, things will get too unstable
otherwise. Should be
later substituted with an EWMA [1].

--------------

Further Ideas for Optimizations
===============================

Resend OGMs with flags etc. of newest OGM
-----------------------------------------

An accepted router with the highest sequence number has the most
up-to-date information about 'Flags', 'Gateway Flags' (and also TT \*?),
they do not have to be stored once per router, once per originator entry
is enough. And any rebroadcasted OGM could update these fields from the
most-up-to-date router. Might make things a little more "complex".

Also aggregate different packet types
-------------------------------------

For instance NDP + OGMs to reduce number of packets sent.

Positive Feedback OGM rebroadcasting
------------------------------------

When for any originator thes compared to the link quality used for the
last rebroadcasted OGM, resend the same OGM but with the path TQ
multiplied with the new, better link TQ value instead.

No OGMs if no hosts
-------------------

To be able to reduce the overhead by just putting some intelligent
"repeater" nodes somewhere without them sending their own OGMs

OGM forwarding optimizations in asymmetric neighborhoods
--------------------------------------------------------

An asymmetric link can either mean that (a) TQ >> RQ or (b) TQ << RQ.
For OGMs only (b) causes trouble and reduces the propagation time of
an OGM
and would therefore be the case to optimize.

A node with a neighborhood with asymmetric links as in case (b) can be
further
devided in the following three cases:

-  Church-Tower-Scenario: A node in the valley has symmetric links to
   most of its close neighbors.
   However there might be a few or a single, distant neighbor with an
   asymmetric link TQ << RQ
   where the same node is receiving fine from but cannot forward packets
   that well to.
-  "Valley"-Scenario: A node is surrounded by nodes with a higher
   transmit power, therefore
   for most of its neighbors the link is asymmetric with TQ << RQ.
-  Or mixtures

The scenarios could be detected via NDP.

Extra OGM Unicasting
~~~~~~~~~~~~~~~~~~~~

In the Church-Tower-Scenario extra OGMs could be forwarded via unicast
to the few nodes.
(Extra OGM broadcasts would be unfair for the other neighbors)

Extra OGM Broadcasting
~~~~~~~~~~~~~~~~~~~~~~

For the "Valley"-Scenario an OGM could be broadcasted more than once.
(Extra OGM Unicasting might result in too many packets)

Resources
=========

* :download:`batman-v5-beweis.tex`
