..
  Copyright 2016, Canonical UK

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=========================
SR-IOV networking support
=========================

SR-IOV is a mechanism for passing hardware virtualized network functions (VF)
or full physical function (PF) directly to KVM instances; the OpenStack
charms should be updated to support this capability.

Problem Description
===================

Using Open vSwitch and the chain of bridges, veth pairs and tap devices incurs
an overhead on network throughput and latency that is not acceptable in some
use cases for OpenStack.

SR-IOV allows part of (a VF) or a full (a PF) SR-IOV enabled network card to be
connected directly to a KVM instance via the virtualization layer provided
by libvirt.

Support for both of these types of passthrough was implemented in full in the
Mitaka release.

Proposed Change
===============

Supporting SR-IOV will require changes in three charms; specifically:

- neutron-api: Enablement of the ML2 mechanism driver for SR-IOV
- nova-cloud-controller: Enablement of appropriate scheduler filters for
  management of PCI passthrough devices
- nova-compute: White listing of PCI devices for use by compute instances.

SR-IOV will be enabled using a configuration option on the neutron-api charm
'enable-sriov'; The nova-cloud-controller charm will be notified of the
state of this option via the relation between the nova-cloud-controller charm
and the neutron-api charm.

In the first implementation, a pci-device-whitelist configuration option
will be provided by the nova-compute charm to allow allocation of specific
PCI devices to compute instances.  This is a direct pass through to a Nova
configuration option.  Later revisions of this feature may use as-yet
unimplemented features in Juju to manage PCI device allocation at the unit
level.

Initial SR-IOV support will be agent-less (i.e. not using the
neutron-sriov-agent).

Flat and VLAN networking modes will be supported.

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  james-page

Gerrit Topic
------------

Use Gerrit topic "sriov-support" for all patches related to this spec.

.. code-block:: bash

    git-review -t sriov-support

Work Items
----------

Initial charm support for SR-IOV
################################

- neutron-api: Add enable-sriov config option, enable mechanism driver, pass
  details to nova-cloud-controller charm.

- nova-cloud-controller: Add PciPassthroughFilter to scheduler filters when
  enable-sriov is enabled by the neutron-api charm.

- nova-compute: Add pci-device-whitelist configuration option, pass directly
  into nova.conf configuration file template(s).

Mojo specification for SR-IOV base enablement
#############################################

- Functional testing specification for deployment of an SR-IOV enabled
  OpenStack Cloud.

SR-IOV VF/PF scheduling testing
###############################

- Focussed testing on ensuring that scheduling behaviour is tuned appropriately
  when SR-IOV is in use within a cloud.

Juju device binding support
###########################

- Stretch objective for Newton cycle; superceeds pci-device-whitelist option
  on nova-compute charm

Repositories
------------

No new git repositories required.

Documentation
-------------

Updates to the README's in the impacted charms will be made as part of this
change.

Security
--------

No additional security concerns.

Testing
-------

Code changes will be covered by unit tests; functional testing will be done
using a Mojo specification.

Dependencies
============

- OpenStack Mitaka.

- SR-IOV enabled hardware in the Ubuntu OpenStack QA lab for functional
  testing

- Juju device management support
