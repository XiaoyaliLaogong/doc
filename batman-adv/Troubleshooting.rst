.. SPDX-License-Identifier: GPL-2.0

B.A.T.M.A.N.-Adv Troubleshooting
================================

Below you can find some problem/solution stories. If you got other
(common) questions about batman-adv, please check the :doc:`Frequently asked question <Faq>` page.

B.A.T.M.A.N.-Adv does not work as expected. What can I do now?
--------------------------------------------------------------

**Q:** Batman-adv isn't working at all or doesn't work as expected?

**A:** First, try to minimize the complexity of your setup, i.e. just
try to build a simple mesh network between two devices (using LAN).
Usually just following the :doc:`quick-start-guide <Quick-start-guide>` is
a good way to start with.
Check also the `Batman
Documentation <https://www.kernel.org/doc/Documentation/networking/batman-adv.txt>`__
at kernel.org.

If you still can't get any setup working at all, go through this
checklist:

Any warnings or errors in the kernel log?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Check dmesg/logread/syslog for warnings, errors or kernel oopses. If you
are seeing a kernel oops, write a `report
ticket </projects/batman-adv/issues/new>`__ and post the log there,
describe your setup and hardware, and ideally how this crash could be
reproduced. If the module displays an error, see go to the next heading.

If the kernel log is just showing cryptic numbers it will be difficult
to help you. You can increase the chances of finding the bug by enabling
the kernel symbol table which will translate these numbers to function
names that can help developers to see what is happening. Please check
the "advanced" section of :doc:`this article <Building-with-openwrt>` to
learn how to enable this functionality on OpenWRT.

Batman-adv is giving me an error when loading the module?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It says: *ERROR: could not insert module batman-adv.ko: Unknown symbol
in module*?

Check dmesg logging output (using dmesg command). If it says 'Unknown
symbol crc32c'. First load the libcrc32c module, using: sudo modprobe
libcrc32c. Installing the module (sudo make install) will fix this
problem as well. As it will check for dependencies using modprobe.

Are both nodes having the same cell id?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Check with 'iwconfig'. Some wifi drivers are a little buggy and do not
always merge two ad-hoc cells, therefore you're usually best advised to
choose and configure one manually (i.e. 'iwconfig wlan0 ap
02:XX:XX:XX:XX:XX'). While configuring a cell-id manually, you should
set the 7th bit of the first byte - or start it with "02:" in other
words. To keep this id (mostly) unique, using one of the routers
mac-address for the rest is usually the safest way to go.

Is 'batctl if' showing the used interfaces as active?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If batman-adv is saying "inactive" for one of the mesh-port interfaces,
make sure this one is up (i.e. 'ip link set up dev wlan0')

Is 'batctl o' showing the other node?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Make sure, that they see each other with a reasonable TQ quality (i.e. >
200). Be aware, that a full TQ update in a dynamic environment but still
with the default ogm-interval setting can be delayed for up to 64
seconds. If they are not seeing each other, check your wifi settings
twice, try ad-hoc meshing without batman-adv or any bridges first to
make sure, that the ad-hoc wifi layer itself works fine and allows any
kinds of packets (ad-hoc mode is usually not the best implemented and
well tested wifi mode in a lot of wifi drivers).

Do all nodes run the same batman-adv version ?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If 'batctl o' does not show the neighbor you expect to see you should
verify whether or not all nodes runt he same batman-adv version. Having
the same version on all nodes is the safest way to be sure that the
versions are compatible. A new release might change the compatibility
number to avoid problems when incompatible versions run in the same
mesh. Incompatible nodes will simply ignore each other. Consult our
:doc:`compatibility table <Compatversion>` to find out which release(s)
carry which compatibility number.

Are those tq-values rather stable or acting crazy?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If they are acting crazy, check if the mesh-port interfaces used by
batman-adv all have a different mac addresses. Otherwise this could
break the routing algorithm in some scenarios.

Does a ping to the other node via the mesh work?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Set static ipv4-addresses (or (autoconfigured) ipv6 addresses) not being
used on any other interfaces on bat0 and check if those two nodes can
ping each other. If this works, but large pings do not (i.e. ping -s
1700), then check the MTU settings. All mesh hosts need an MTU of 24
Bytes less (i.e. on bat0 or any host bridged into the mesh) than on
their mesh-port interfaces' MTU (the interfaces you've added via 'batctl
if'). For instance you could chose to increase the MTU on all mesh-port
interfaces to 1524 (or decrease it on all hosts, bat0 or any host
interface being bridged into the mesh, to 1476 which is usually harder
to maintain when having 'foreign' hosts).

Does 'batctl ping' to the other node work?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you can see the other node but can't send any packets through the
mesh network, check whether you can ping it with batman-adv's internal
echoing packets (see 'batctl ping [STRIKEOUT:h' for usage info). If this
works, check your layer 3 settings, in general your routes and IP
adresses] don't use the same routes/addresses on different interfaces,
don't set any ip-addresses on the mesh-port interfaces. Just use bat0 or
the bridge you might have created on top of it.

Are the wifi/lan LEDs blinking like crazy?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Make sure you DON'T bridge any mesh-port-interfaces with your bat0
interface!
