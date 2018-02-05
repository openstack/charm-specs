..
  Copyright 2018, Canonical UK

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===============================
Routed Provider Networks for Neutron Charms
===============================

As L3-oriented fabrics get more widely deployed in DCs there is a need to
support multi-segment networks in Neutron charms. Routed provider networks
were implemented in Neutron itself in order to model deployments with multiple
isolated L2 networks each with their own VLANs (switch fabrics) and network
or compute nodes attached to those networks.

Problem Description
===================

Deployments that have machines connected to separate switch fabrics and hence
provider networks need to be aware of multiple subnets attached to different
L2 segments. While VLAN IDs on different switch fabrics may match, they
define isolated broadcast domains and should have different subnets
used on them for proper addressing and routing to be implemented.

Provider Networks in Neutron used to have a single segment but starting with
`Newton <https://specs.openstack.org/openstack/neutron-specs/specs/newton/routed-networks.html>`__ 
there is support for multi-segment networks which are called routed provider
networks. From the data model perspective a subnet is associated with a
network segment and a given provider network becomes multi-segment as a
result of subnet to segment assignments. This concept is the same as creating
a Virtual Routing and Forwarding (VRF) instance because end hosts on an L3
network are assumed to be in a single address space but possibly different
subnets and routing needs to be implemented to make sure all hosts and
instances are mutually L3-reachable.

Considerations for routed provider networks include:

* scheduling (Neutron needs to use Nova Placement API);
* subnets become segment-aware;
* ports become segment-aware;
* IP address to port assignment is deferred until a segment is known;
* placement with IP address exaustion awareness;
* live migration and evacutation of instances (constrain to the same L2);
* DHCP and meta-data agent scheduling.

A `nova spec <http://specs.openstack.org/openstack/nova-specs/specs/newton/implemented/neutron-routed-networks.html
>`__ for routed provider networks contains a detailed overview of the
implemenation details to address the above points, including usage of
`generic resource pools. <http://specs.openstack.org/openstack/nova-specs/specs/newton/implemented/generic-resource-pools.html>`__

In order to account for IPv4 address ranges associated with segments Neutron
uses Nova placement API to create IPv4 inventories and update them if needed.
Neutron also creates a Nova host aggregate per segment.

There are several points to consider when it comes to charm modifications to
support this functionality:

* "segments" service plugin enablement (unconditionally or via an option);
* Deployment scenarios in conjunction with other features (L3 HA, DVR, BGP);
* Floating IP usage;
* Impact on existing deployments (upgrades).

Also, from the Charm and Juju perspective the following needs to be addressed:

* Handling of physnet mappings on different switch fabrics using Juju AZs or
  tag contraints;
* Appropriate placement of units to make sure necessary agents are deployed.

Deployment Scenarios
+++++++++++++++++++++++

Deployment scenarios should include both setups with charm-neutron-gateway and
without it. Moreover, for the use-case where charm neutron-gateway is present
we need to account for DVR and L3 HA cases.

* provider networks at networks nodes only (SNAT and FIP sub-cases);
* `provider networks on compute nodes without dedicated network nodes; <https://github.com/openstack/neutron/blob/master/doc/source/admin/deploy-ovs-provider.rst>`__
* `OVS HA using VRRP; <https://github.com/openstack/neutron/blob/master/doc/source/admin/deploy-ovs-ha-vrrp.rst>`__
* `DVR and L3 HA using VRRP; <https://github.com/openstack/neutron/blob/master/doc/source/admin/config-dvr-ha-snat.rst>`__
* `Deployments with dynamic routing agents; <https://github.com/openstack/neutron/blob/master/doc/source/admin/config-bgp-dynamic-routing.rst>`__

Proposed Change
===============

Given that there are certain limitations in how routed provider networks are
implemented in Newton release this spec is targeted at Ocata release.

Segments Service Plugin
+++++++++++++++++++++++

`A related Ocata documentation section <https://docs.openstack.org/ocata/networking-guide/config-routed-networks.html#example-configuration>`__ contains 3 major configuration requirements: 

* "segments" needs to be added to the service_plugins list (neutron-api);
* A compute placement API section needs to be added to neutron.conf;
* Physical-network related config options on gateway and compute nodes need to
  match a given switch fabric configuration (see a section on that below);

The proposed approach, therefore, is as follows:

* Unconditionally render "segments" service_plugin in charm-neutron-api if a
  deployed OpenStack version is Ocata or newer;
* Add Nova placement API section rendering to charm-neutron-api if a deployed
  OpenStack version is Ocata or newer.

The reason for unconditional addition of the segments service plugin is due to
how feature is implemented and documented.

Unless a `subnet is associated with a segment <https://developer.openstack.org/api-ref/network/v2/#create-subnet>`__, a provider network is considered
to be single-segment and old behavior applies. There is also a requirement
that either all subnets associated with a network need to have a segment or
none of them must have one. This is documented both in `public documentation <https://docs.openstack.org/neutron/pike/admin/config-routed-networks.html>`__
and in `code. <https://github.com/openstack/neutron/blob/49d614895f44c44f9e1735210498facf1886c404/neutron/services/segments/exceptions.py#L26-L29>`__

        The Networking service enforces that either zero or all subnets on a
        particular network associate with a segment. For example, attempting
        to create a subnet without a segment on a network containing subnets
        with segments generates an error.

This distiction is what allows unconditional addition of this service plugin
for the standard ML2 core plugin.

Multiple L2 Networks (Switch Fabrics)
+++++++++++++++++++++++

Each network segment includes `network type information <https://github.com/openstack/neutron/blob/3e34db0c19cdcc86cd5f1b72d6374c3eca0faa7e/neutron/db/models/segment.py#L30-L54>`__ this means that for ML2 plugin flat_networks and network_vlan_ranges
options there may be different values depending on a switch fabric. In this
case, there need to be multiple copies of the same charm deployed as
applications with different names to create per-fabric configurations. This
can be modeled in Juju in terms of availability zones although this mapping
is not exact because there may be several availability zones on the same
switch fabric or there may be a fabric per availability zone. In a general
case, tags contraints can be used in Juju to provide placement metadata
related to switch fabrics. Either way, to achieve the separation of config
namespaces the following charms may need per-fabric configuration:

* charm-neutron-openvswitch (subordinate, provider networks or DVR cases);
* charm-nova-compute (as a primary for neutron-ovs charm - defines placement);
* charm-neutron-gateway (in scenarios where it is deployed).

Neutron charms (ovs or gateway) contain the following fabric-specific options:

* bridge-mappings;
* flat-provider-networks;
* vlan-ranges;
* enable-local-dhcp-and-metadata (needs to be true everywhere);

Albeit it would be tempting to assume that this configuration will be the same
for all nodes in a given deployment and that switches the nodes will be
connected to will have identical configuration, we need to account for a
general case.

Multi-application approach allows to avoid any charm modifications besides
charm-neutron-api and have the following content in bundle.yaml::

        variables:
          data-port:           &data-port           br-data:bond1
          vlan-ranges:         &vlan-ranges         provider-fab1:2:3 provider-fab2:4:5 provider-fab3:6:7
          bridge-mappings-fab1: &bridge-mappings-fab1 provider-fab1:br-data
          bridge-mappings-fab2: &bridge-mappings-fab2 provider-fab2:br-data
          bridge-mappings-fab3: &bridge-mappings-fab3 provider-fab3:br-data
          vlan-ranges-fab1:     &vlan-ranges-fab1     provider-fab1:2:3
          vlan-ranges-fab2:     &vlan-ranges-fab2     provider-fab2:4:5
          vlan-ranges-fab3:     &vlan-ranges-fab3     provider-fab3:6:7

        # allocate machines such that there are enough
        # machines in attached to each switch fabric
        # fabrics do not necessarily correspond to
        # availability zones
        machines:
          "0":
            constraints: tags=compute,fab1
          "1":
            constraints: tags=compute,fab1
          "2":
            constraints: tags=compute,fab1
          "3":
            constraints: tags=compute,fab1
          "4":
            constraints: tags=compute,fab1
          "5":
            constraints: tags=compute,fab2
          "6":
            constraints: tags=compute,fab2
          "7":
            constraints: tags=compute,fab2
          "8":
            constraints: tags=compute,fab2
          "9":
            constraints: tags=compute,fab3
          "10":
            constraints: tags=compute,fab3
          "11":
            constraints: tags=compute,fab3
          "12":
            constraints: tags=compute,fab3
        services:
          nova-compute-kvm-fab1:
            charm: cs:nova-compute
            num_units: 5
            bindings:
            # ...
            options:
            # ...
              default-availability-zone: az1
            to:
            - 0
            - 1
            - 2
            - 3
            - 4
          nova-compute-kvm-fab2:
            charm: cs:nova-compute
            num_units: 5
            bindings:
            # ...
            options:
            # ...
              default-availability-zone: az2
            to:
            - 5
            - 6
            - 7
            - 8
          nova-compute-kvm-fab3:
            charm: cs:nova-compute
            num_units: 5
            bindings:
            # ...
            options:
            # ...
              default-availability-zone: az3
            to:
            - 9
            - 10
            - 11
            - 12
          neutron-openvswitch-fab1:
            charm: cs:neutron-openvswitch
            num_units: 0
            bindings:
              data: *overlay-space-fab1
            options:
              bridge-mappings: *bridge-mappings-fab1
              vlan-ranges: *vlan-ranges-fab1
              prevent-arp-spoofing: True
              data-port: *data-port
              enable-local-dhcp-and-metadata: True
          neutron-openvswitch-fab2:
            charm: cs:neutron-openvswitch
            num_units: 0
            bindings:
              data: *overlay-space-fab2
            options:
              bridge-mappings: *bridge-mappings-fab2
              vlan-ranges: *vlan-ranges-fab2
              prevent-arp-spoofing: True
              data-port: *data-port
              enable-local-dhcp-and-metadata: True
          neutron-openvswitch-fab3:
            charm: cs:neutron-openvswitch
            num_units: 0
            bindings:
              data: *overlay-space-fab3
            options:
              bridge-mappings: *bridge-mappings-fab3
              vlan-ranges: *vlan-ranges-fab3
              prevent-arp-spoofing: True
              data-port: *data-port
              enable-local-dhcp-and-metadata: True
        # each of the apps needs to be related appropriately
        # ...

The above bundle part is for a setup without charm-neutron-gateway, although
it can be added easily using the same approach. Given that there are no
relations between charm-neutron-gateway and charm-neutron-openvswitch there
is no problem with relating per-fabric services. What has to be kept in mind
is the infrastructure routing for overlay networks on different fabrics so
that VXLAN or other tunnels can be created between endpoints.

Documented Requirements and Limitations
+++++++++++++++++++++++

The Ocata Routed Provider Networks `guide mentions scheduler limitations, <https://docs.openstack.org/ocata/networking-guide/config-routed-networks.html#limitations>`__
however, they are made in reference to Newton and are likely outdated. Looking
at the `Newton documentation <https://docs.openstack.org/newton/networking-guide/config-routed-networks.html>`__ it is certain that this documentation section was
not updated for Ocata.

Minimum API version requirements are present in the Pike `release documentation <https://github.com/openstack/neutron/blame/stable/pike/doc/source/admin/config-routed-networks.rst#L144-L148>`__

Floating IP usage is not possible with routed provider networks at the time
of writing (Pike) due to the fact that a routed provider network cannot be
used with `external-net extension. <https://developer.openstack.org/api-ref/network/v2/#external-network>`__

This might be considered a serious limitation, however:

* Deployments that rely on routed provider networks are inherently L3-oriented
  and it is likely that NAT functionality will not be required that often
  anyway alleviating the need for floating IPs;
* Routed provider networks are not enforced at deployment time.

An `RFE <https://pad.lv/1667329>`__ exists to address this limitation based on
`subnet service types <https://specs.openstack.org/openstack/neutron-specs/specs/newton/subnet-service-types.html>`__ and `dynamic routing <https://docs.openstack.org/newton/networking-guide/config-bgp-dynamic-routing.html>`__.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  dmitriis

Gerrit Topic
------------

Use Gerrit topic "1743743-fe-routed-provider-networks" for all patches related
to this spec.

.. code-block:: bash

    git-review -t 1743743-fe-routed-provider-networks

Work Items
----------

* add "segments" to service_plugins in charm-neutron-api for Ocata+;
* add section-placement to charm-neutron-api and import it in Ocata+ templates

Repositories
------------

No new repositories.

Documentation
-------------

Release notes should mention that this functionality can be used.

It might be worthwhile to create a dedicated guide to cover how different
OpenStack network deployment scenarios map to charm options.

Security
--------

No apparent security risks.

Testing
-------

The following functional test can be introduced even on a single fabric but
with different VLANs used to simulate separate fabrics and L3 connectivity:

* deploy a bundle with two neutron-openvswitch and nova-compute charms and
  configure them with different provider network settings as described above;
* create a multi-segment network as described `in the documentation; <https://docs.openstack.org/neutron/pike/admin/config-routed-networks.html#create-a-routed-provider-network>`__
* create several instances with interfaces attached to the routed provider
  network created above and make sure that L3 connectivity is provided
  externally between segments;
* verify L3 connectivity between instances.

Dependencies
============

None
