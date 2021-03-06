.. SPDX-License-Identifier: GPL-2.0

Client announcements
====================

Each mesh protocol offering network access to non-mesh clients needs a
'client announcement' system. The task of such a system is to inform
every mesh node in the network of any connected non-mesh client, in
order to find the path towards that client from any given point in the
mesh. A client has an identifier (usually its IP address for layer 3
routing and MAC address for layer 2 routing) that is flooded through the
mesh. As B.A.T.M.A.N.-Advanced is a Layer2 mesh routing protocol clients
are represented by their MAC address.

The local translation table
---------------------------

Every client MAC address that is recognized through the mesh interface
will be stored in a node local table called "local translation table"
which will contain all the clients the node is currently serving. This
table is the information a node has to spread among the network in
order to make its clients reachable. This is because when a node wants
to contact a particular client, thanks to this information, it knows
the originator it has to send the data to.
Each node local table has a particular attribute: the translation
table version number (ttvn). The value of this attribute represents
the version of the table that is incremented by one each time the
local table changes (a client has been added/removed). For
optimization reasons all changes which happen within the same OGM
interval are aggregated into a single ttvn increment.
Every OGM broadcast contains the current ttvn and a set of CRC32
checksum values of the local table to allow the receiving nodes to
quickly decide whether the tables are in sync or not.

The need of including more than one CRC32 value is due to the fact that
the local translation table is aware of which VLAN each client belongs
to and to ensure that all the mechanisms in batman-adv work as expected,
a single CRC32 value per VLAN is required. Therefore each node computes
as many CRC32 values as the number of detected VLANs.

The global translation table
----------------------------

Every node in the network has to store all the other node's local
tables. To achieve this, another table is needed: the "global
translation table". It is a set of entries where each contains the
client MAC address and a pointer to the originator that is currently
announcing it.

Updating the tables
-------------------

At boot time, every node will have an empty local table and empty global
one. Its ttvn will be initialized to 0. The OGM broadcast represents the
local table propagation event.

As soon as a local event occurred (client added/deleted) the ttvn is
incremented by one on the next propagation event (OGM broadcast). In
addition, the local changes since the last OGM broadcast are appended to
the OGM itself. This mechanism helps to avoid the more expensive table
request operation (see below) as any receiving node can retrieve the
changes from the OGM to update its global translation table.

The tt change entry is composed by the following fields (to have a
better understanding of how this entries are propagated on the wire,
please have a look at the :ref:`TT TVLV container description <batman-adv-TVLV-Translation-table-messages>`):

-  Flags: Indicates whether this client address is to be added or
   removed.
-  Reserved (3 bytes): needed to make the structure size a multiple of
   32bits
-  Client Address: Address of the concerning client.
-  VLAN Identificator: ID of the VLAN where this client is connected to

::

      0                   1                   2                   3
      0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |    Flags      |              Reserved                         |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |                     Client Address...                         |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
     |     ...Client Address         |          VLAN ID              |
     +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

These changes are also appended to the 2 following OGMs if there were no
new changes in the meantime, thereby increasing the chance for neighbor
node to receive the changes in the event of packet loss. If a node
receives an OGM containing a new ttvn without the changeset (e.g. it
missed all 3 OGMs with the changeset or the changeset exceeded the
packet size) or any of the CRC32 checksums does not match it can issue a
table request. In particular a node can ask for two different
information: either the changeset of the current ttvn or the full local
table.

A TT request packet contains a :ref:`TT TVLV container <batman-adv-TVLV-Translation-table-messages>` with a particular flag set telling its type. As a
request it does not have any change appended.

The reply to any TT request can either be the changeset of the current
ttvn or the full local table. The changeset always is the preferred
format to answer the request as it comes with a smaller overhead
penalty. In some cases a changeset reply is not possible (e.g. a node
just joined the mesh) which makes a full table reply necessary. If the
reply contains the changeset of the current ttvn the corresponding
change entries are appended to the reply. If the reply contains the full
table the entire list of clients is appended to the message using the
change entry format containing the MAC addresses and corresponding
flags.

Like the request a TT reply packet contains a
:ref:`TT TVLV container <batman-adv-TVLV-Translation-table-messages>` with a flag
saying it is a reply and another one indicating if it carries a full
table or just the last changeset. All the change entries are then
appended to the message.

Table request forwarding:
-------------------------

To reduce the overhead of the table request operations, each node on the
path of a TT request message will inspect the content and decide whether
it has the correct information to answer the request directly without
forwarding the the message any further. The ttvn and the CRC32 checksums
contained in the request message provide sufficient means to verify if
it matches with the locally available information. If they match the
node can directly reply to the request (with the full table of the
destination's translation table or the changeset of the last ttvn if
possible). If something didn't match, the node will forward the packet
to the nexthop in the path to the destination.

Computing the CRC32 checksum:
-----------------------------

Since batman-adv-2014.0.0 the CRC32 checksum is computed on a VLAN
basis. It means that the local translation entries are divided in
several groups, based on the VLAN they are connected to, and for each of
them a different checksum is computed. This VLAN distinction is required
to keep the table checking mechanism effective also when a node decides
to filter out entries for a given VLAN.

An example is the Bridge Loop Avoidance mechanism that forces backbone
nodes to skip global entries advertised by other backbone nodes bridged
with the VLAN where such nodes are in touch with each other (check the
:doc:`Bridge Loop Avoidance page <Bridge-loop-avoidance-II>` for more
details).

The checksum values which are sent along with the ttvn field in the OGM
(or TT request/responde messages) are computed for a generic originator
O as the XOR of the CRC32 values of the tuple { MAC address, VID, sync
flags } of each local translation table entry.

**Pseudocode**:

::

    tt_local_crc(orig_node O, vlan_id ID) {
    res = 0;
    for each tt_local_entry in vlan ID:
         tmp = crc32(0, tt_local_entry->vid)
         tmp = crc32(tmp, tt_local_entry->addr)
         tmp = crc32(tmp, tt_local_entry->sync_flags)

         res = res XOR tmp
    endfor
    return res

Improving data routing
----------------------

The ttvn field has also been added to the unicast packet header. A node
sending a packet of this type will set this field to the currently known
destination's ttvn. Along the path from the source to the destination,
every node will inspect the packet and check whether it knows an higher
ttvn for the same destination; if so, the node will look in its global
translation table to see which is the current mesh node serving the
client which the packet is directed to. At this point the intermediate
node will replace the destination and the ttvn values in the unicast
packet header and will re-forward the packet to the new destination
(possibly the same).

This behavior slightly helps in case of roaming: a client moved from a
mesh node to another, but the source node doesn't know this change yet.
Data is sent to the old node serving the client, but as soon as the
packet reaches an updated node, it will be redirected to the new
(possibly correct) destination.

Reducing client joining latency
-------------------------------

Upon connecting, a client has to wait to be announced to the rest of the
mesh network before being able to communicate with any other host. The
average delay introduced by this step varies depending on the originator
interval value set on the node which serves the new client.

To face this issue the TranslationTable component introduced a new
feature called **SpeedyJoin**. This feature enables nodes in the network
to add a temporary route towards not yet announced client but which they
have already got a packet *broadcast* from. The constraint of receiving
a broadcast packet is due to the fact that this type is the only one (up
to one) that contains the address of the source node, that is needed in
order to add a new route towards the client. However, this is not a
problem, because a newly joining client is likely to issue a DHCP or an
ARP Request (usually to detect the gateway MAC address) upon connection.

The new entry is added to the global TranslationTable and marked as
*temporary* with a special flag (BATADV\_TT\_CLIENT\_TEMP) until a node
claims it with the classic announcement mechanism. If none of the node
belonging to the mesh network will announce the temporary client, the
latter will be deleted upon a timeout expiration set for the purpose.

Limitations
-----------

-  Too many local clients: The size of the local translation table
   depends on the number served clients. This size cannot exceed the
   maximum **fragmented** packet size and if the limit is reached, new
   clients are ignored. This is a virtual value given by the smallest
   MTU among all the hard-interfaces in use multiplied by the maximum
   number of allowed fragments (default to 16). This means that at
   compile time the user could potentially increase the number of
   fragments a node can send, thus increasing the local translation
   table maximum size.

Notes
-----

A research project has been done on this topic and it is freely
available here: https://eprints.biblio.unitn.it/2269/.
