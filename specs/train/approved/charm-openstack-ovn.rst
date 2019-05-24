..
  Copyright 2018 Aakash KT
  Copyright 2019 Canonical

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===
OVN
===

OVN provides open source network virtualization for Open vSwitch (OVS):

- Logical switches

- IPv4 and IPv6 logical routers

- L2/L3/L4 ACLs (Security Groups)

- Multiple tunnel overlays (Geneve, STT and VXLAN)

- Logical L4 load-balancing

- TOR-based L2 logical-physical gateways (VTEP)

- Software-based L2/L3 logical-physical gateways


OVN integration components exists for OpenStack Neutron, Kubernetes and
others.


Problem Description
===================

There is currently no charms available that lets you easily deploy the
services required to make use of OVN.

This will also benefit OPNFV's JOID installer in providing another scenario in
its deployment.

Proposed Change
===============

Implement new reactive charms that adds support for deploying OVN and
integration with OpenStack.

Alternatives
------------

None

Implementation
==============

The implementation consists of the following charms:

- ovn

  - Principal charm that deploys ``ovn-northd``, the OVN central control
    daemon, and ``ovsdb-server``, the Open vSwitch Database (OVSDB).

  - The ``ovn-northd`` daemon is responsible for translating the high-level
    OVN configuration into logical configuration consumable by daemons such
    as ``ovn-controller``.

  - The ``ovn-northd`` process talks to OVN Northbound- and Southbound-
    databases.

  - The ``ovsdb-server`` exposes endpoints over relations implemented by the
    ``ovsdb`` interface.

  - The charm supports clustering of the OVSDB, you must have a odd number of
    units for this to work.  Note that write performance decreases as you
    increase the number of units.

  - Running multiple ``ovn-northd`` daemons is supported and they will operate
    in active/passive mode.  The daemon uses a locking feature in the OVSDB to
    automatically choose a single active instance.

  - The charm MUST configure TLS transport between OVSDB and its clients
    supporting both charm static configuration or with help from ``vault`` and
    the ``tls-certificates`` interface.

- ovn-controller

  - Subordinate charm that deploys the OVN local controller and Open vSwitch
    Database and Switch.

  - It connects up to the OVN Southbound database over the OVSDB protocol, down
    to the unit local Open vSwitch database over the OVSDB protocol and to the
    unit local Open vSwitch daemon via OpenFlow.

  - Each hypervisor, software gateway or other units in need of OVN networking
    runs its own independent copy of ``ovn-controller``.

  - Modules to integrate with specific software, such as OpenStack
    ``nova-compute`` and Kubernetes ``worker``, should all be added to this
    subordinate charm.  Which module(s) to activate should be determined at
    runtime through presence of optional relations.

  - A LXD profile will be required for this charm to enable container
    placement of OVN/OVS components.

- ovn-dedicated-controller

  - Principal charm for deployments that require a dedicated software gateway.

  - This charm will be built from the ``ovn-controller`` subordinate codebase.

- neutron-api-plugin-ovn

  - Subordinate charm that deploys the ``networking-ovn`` component on
    ``neutron-api`` units and augments Neutron's configuration for use with
    the OVN ML2 plugin.

- octavia-ovn-provider

  - Subordinate charm that deploys the ``networking-ovn`` component on
    ``octavia`` units and augments Octavia's configuration for use with the
    OVN provider driver.

The implementation consists of the following interfaces:

- ovsdb

  - Interface that facilitates a provider charm to publish connection
    properties of a OVSDB.

  - Interface that facilitates a requirer charm to consume a remote OVSDB.

- octavia-provider

  - Interface that facilitates a provider subordinate charm to publish
    information about a Octavia provider driver to and to trigger restart of
    the Octavia service.

  - Interface that facilitates the Octavia charm to add provider information
    and react to restart requests from a subordinate provider charm.

The charms should be implemented in such a way so that the non-OpenStack
specific components can be deployed with other software components such as the
Charmed Distribution of Kubernetes.

Wherever possible and relevant, existing interfaces or charm libraries should
be consumed or extended to work with the new charms.

Assignee(s)
-----------

Primary assignee:
  fnordahl

Gerrit Topic
------------

Use Gerrit topic "ovn-charm" for all patches related to this spec.

.. code-block:: bash

    git-review -t ovn-charm

Work Items
----------

See Implementation_ section

Repositories
------------

- charm-ovn

- charm-ovn-controller

- charm-ovn-dedicated-controller

- charm-neutron-api-plugin-ovn

- charm-octavia-ovn-provider

- charm-interface-ovsdb

- charm-interface-octavia-provider

Documentation
-------------

Each charm should contain a README with instructions on deploying the charm.

In addition to that a appendix will be added to the deployment guide with
details about how to deploy OpenStack with OVN.

Security
--------

Communication between OVSDB and its clients is authenticated and secured by the
use of TLS and PKI.  This must be implemented using secure best practices.

TLS configuration should be possible both statically through configuration and
through automatic certificate provisioning with ``vault`` and the
``tls-certificates`` interface.

Testing
-------

Code written or changed will be covered by unit tests; functional testing will
be implemented using the ``Zaza`` framework.

Dependencies
============

- There may be need for changes to the layout of the ``openvswitch`` source
  package to cater for running the ovsdb-server process without having the
  package attempting to configure the switching components.  This will help
  with container placement of the ``ovsdb`` charm without requiring a LXD
  profile.

- There may be need for changes to the layout of the ``netwokring-ovn`` source
  package to cater for Octavia provider driver enablement.
