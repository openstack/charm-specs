..
  Copyright 2016 Canonical
  
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
External Networking REDUX
=========================

Refactor neutron-gateway and neutron-openvswitch charms to support more
flexible configuration of 'external' north/south traffic routing to
tenantn networks.

Problem Description
===================

The neutron-gateway and neutron-openvswitch charms currently support the
'ext-port' configuration option to plumb the Open vSwitch br-ex bridge
into the external/public network.

OpenStack switched to a more flexible way of doing this a few cycles back,
and the use of the default 'br-ex' == external network approach is
officially deprecated.

We need to refactor the charms to support the reference way of configuring
external networking, which provides more flexibility than the current
approach but does require more knowledge of Neutron.

Use Cases
---------

Using the reference approach to external networking, its possible to have
multiple external networks served from the same set of neutron gateway
units, potentially on the same physical network with VLAN segementation:

    neutron net-create --provider:network_type vlan \
                       --provider:segmentation_id 400 \
                       --provider:physical_network physnet1 --shared external

    neutron net-create --provider:network_type vlan \
                       --provider:segmentation_id 401 \
                       --provider:physical_network physnet1 --shared \
                       --router:external=true floating

    neutron router-gateway-set provider floating


with the associated neutron-gateway configuration:

    neutron-gateway:
        bridge-mappings:         physnet1:br-data
        data-port:               br-data:eth1

Proposed Change
===============

Both the neutron-gateway and neutron-openvswitch charms will need to be
updated to support this change; for now the 'ext-port' configuration will
be deprecated for backwards compatibility with existing installations
preserving current behaviour.

The existing 'data-port' configuration option will be used instead to
configure bridge-mappings and connectivity to physical network ports
on the units deployed.

Alternatives
------------

No alternatives.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  james-page
  lathiat (Trent Lloyd)

Gerrit Topic
------------

Use Gerrit topic "ext-net-redux" for all patches related to this spec.

.. code-block:: bash

    git-review -t ext-net-redux

Work Items
----------

- Updates for neutron-gateway charm
- Updates for neutron-openvswitch charm
- Updates for openstack-charm-testing to configure external networking
  in the new style.
- Updates to Mojo specifications for new style configuration.
- Updates to official bundles with details on new configuration for
  external networking.

Repositories
------------

None required.

Documentation
-------------

README's in impacted charms will be updated with details on how to
configure external networking.

Security
--------

No additional security requirements.

Testing
-------

Unit tests will be used to validate the code; Mojo specifications will
be updated to use the new style of external networking configuration
along with openstack-charm-testing.

Dependencies
============

- No additional dependencies for this spec.

References
----------

- https://bugs.launchpad.net/neutron/+bug/1491668
- http://docs.openstack.org/mitaka/networking-guide/scenario-classic-ovs.html
