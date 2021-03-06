.. SPDX-License-Identifier: GPL-2.0

Research Proposals on B.A.T.M.A.N.-Advanced
===========================================

Improving link metrics
~~~~~~~~~~~~~~~~~~~~~~

The sheer measurement of the OGM delivery ratio is
a poor metric for link quality evaluation.
It is not able to distinguish fast links from slow ones,
it is not able to tell congested links apart from
unused ones, it is not able to distinguish point-to-point
links from broadcast ones.
Improving the link evaluation metrics without becoming
dependent from the specific link-level technology is however
a challenge. Exploring the role of delay, delay jitters, information
coming from the data plane (for instance in 802.11 data packets are
sent at different link speeds depending on the actual destination
and not at the base speed used for broadcast) can yield results
leading to improved metrics.

Improving multi-radio strategy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In presence of multiple (wireless and wired) interfaces
the present strategies for interface selection in Batman are
based on simple and non-general heuristics, with the
consequence that very often sub-optimal decisions are taken.
To improve these strategies the following fundamental points need
attention and solutions:

* Capability to differentiate between different MAC/PHY layers
  (e.g., a reliable 1Gbit/s optical link from WiFi, or also a
  reliable point-to-point WiFi bridge from a generic multi-point
  WiFi interface).
* Metrics and algorithms for iteratively improving the topology
  based on the above information.

Improving Broadcast communications
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some applications, specially related to network management like
, or to services publish/subscribe models, require efficient
and reliable broadcast/multicast (IP layer) communications, which
are however always mapped onto layer 2 broadcast packets.
Presently batman transmits these packets 3 times, with a large
overhead
and still no guarantee of delivery.
Many solutions can be studied, starting from intelligent re-broadcast
to multiple-unicast flooding.

Improving Multicast communications
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

(partly started, there
is a concept and a partial implementation)
Interacts with the point above ... PIM Sparse/Dense mode can be
analyzed and extended to wireless networks (PIM per-se cannot
work on wireless interfaces).

Define a Neighbor discovery independent protocol
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

(partly started - ELP. There is a partial concept and partial
implementation)

Fast dead link detection or Fast link switching
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Would also improve mobility

Security/Authentication
~~~~~~~~~~~~~~~~~~~~~~~

Adding a security and authentication layer to protect batman
messages from malicious or malfunctioning nodes.

Distributed filtering
~~~~~~~~~~~~~~~~~~~~~

Packet filtering in ad-hoc and mesh networks can help to define
different user profiles, reduce the impact of unwanted traffic and
make the network more robust to attacks. The problem of implementing a
network-wide filtering policy to be enforced in all the mesh nodes
with user-level granularity must still be resolved since the diffusion
of rule-sets and their enforcement has a great impact in a
resource-limited networks both for the generated traffic and the
computation required.
