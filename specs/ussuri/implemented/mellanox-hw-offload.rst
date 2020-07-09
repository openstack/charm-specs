..
  Copyright 2019, Canonical Ltd

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=================================
ML2/OVS Mellanox Hardware Offload
=================================

High end networking adapters such as the Mellanox Connect-X series support
up to 200Gbps networking; instances using traditional software virtualized
networking via OVS and virtio-net cannot achieve networking throughput of
more than 10Gbps without a high CPU load; it’s possible to use SR-IOV to
increase performance to nearer 100Gbps but that has limitations in terms
of the flexibility of networking options (no support for overlay networks)
and no support for security groups.

The Mellanox Connect-X 5+ supports hardware offload via a feature called
ASAP2; integrated with Open vSwitch (OVS), this pushes the network data
path from the kernel directly onto a high performance embedded switch
on the network card.  This will allow instances to achieve much higher network
performance with very low CPU overheads as well as utilizing as much
of the overall network capacity to the hypervisor as possible.

Problem Description
===================

Hardware offload support is not currently enabled in the OpenStack charms.

Proposed Change
===============

The neutron-openvswitch charm will be updated to support configuration of
Mellanox Connect-X 5+ network adapters to support hardware offloading
(card needs to be placed in switchdev mode on boot);  OVS will
be configured to make use of hardware offloading where possible.

A new tool (mlnx-switchdev-boot) will be developed to configure the
Connect-X card in the right mode on boot prior to it being used by OVS
for networking.  This may be superseded at some future date by features
in networkd and/or netplan.

This feature does make use of a number of leading edge features across the
hypervisor software stack:

- Linux Kernel >= 5.2
- OVS >= 2.11.0

This limits the scope of support to OpenStack Stein or later in-conjunction
with the Ubuntu LTS hardware enablement kernel from Ubuntu 19.10.

The Connect-X series of cards also supports Virtual Function (VF) Link
Aggregation (LAG) providing upstream switch/cable/port resilience for
hardware offloaded ports which are supported using a VF on the
underlying network card - this is driven by the normal configuration
process for bonding network devices via Linux - in this case the
Physical Functions (PF’s) for the network cards.

The existing SR-IOV VF configuration code and scripts in the
neutron-openvswitch charm is a little inconsistent and not very reliable
on reboot; this change will cover refactoring the SR-IOV VF code into
a new tool (sriov-netplan-shim) dealing with configuration of VF
functions via the charm and on reboot.  This tool will be superceeded
by SR-IOV support in netplan at some future date.

.. note::

    The most recent kernel and OVS versions don’t yet support connection
    tracking offload which means that security groups cannot be offloaded
    to the network card switch; this is under development and will land in
    a later release of OVS and Linux.

.. note::

    The total number of hardware offloaded ports a hypervisor can support
    is limited by the total number of VF’s that the underlying network
    cards can support.

Alternatives
------------

It’s worth noting that for VLAN or flat networking requirements, it’s
possible to achieve high throughput networking to instances by using
SR-IOV which is already supported by the neutron-openvswitch charm.

Implementation
==============

Assignee(s)
-----------

Primary assignees:
  wgrant
  james-page

Gerrit Topic
------------

Use Gerrit topic "mlnx-hardware-offload" for all patches related to this spec.

.. code-block:: bash

    git-review -t mlnx-hardware-offload

Work Items
----------

Develop and test tool for configuration of Connect-X adapters for hardware
offload support.

Update neutron-openvswitch to support configuration of OVS in hardware
offload mode.

Add appendix to charm deployment guide to detail configuration and use
of OpenStack with Mellanox Connect-X 5+ cards in hardware offload mode.

Repositories
------------

A new repository (mlnx-switchdev-mode) will be required for tooling to
configure the Mellanox Connect-X networking cards on boot.  This tool
has scope outside of the OpenStack charms project so can be developed
on github.

A new repository (sriov-netplan-shim) will be required for tooling to
configure SR-IOV VF functions; this tool will superceed the existing
SR-IOV VF configuration code in the neutron-openvswitch charm.

Documentation
-------------

Documentation of this feature will be provided in the charm deployment guide.

Security
--------

No additional security risks are introduced by this feature.

Testing
-------

Testing of this feature requires specific hardware in the form of Mellanox
Connect-X 5+ networking cards.

Dependencies
============

- Linux >= 5.2
- OVS >= 2.11.0
- Mellanox Firmware >= 16.26.1040
