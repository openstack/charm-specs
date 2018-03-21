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
Neutron Dynamic Routing
===============================

OpenStack supports dynamic routing, specifically BGP. BGP forms a significant
part of the routing infrastructure for many TelCos and large other
organizations with complex networks. Our charms need to grow support for
dynamic routing.


Problem Description
===================

In a typical setup with Neutron Gateway, infrastructure routers do not have
knowledge of the private networks allocated to OpenStack tenants. As a result,
services external to OpenStack do not know how to route traffic to OpenStack
instances. As OpenStack networks are elastic and therefore can evolve in many
ways, a way of communicating changes between OpenStack and external routers is
needed.

Since networks in OpenStack are self serviced, any addition or removal of
OpenStack network should not depend on network administrator’s actions.
Network limitations should be set up front. This is further facilitated by
the use of subnet pools in OpenStack.

Other implications include decoupling subnets from layer 2, allowing different
next-hops for floating IPs.


Proposed Change
===============

Neutron dynamic routing solves this problem by introducing a BGP speaker, which
then peers with the existing infrastructure router(s) and announces the
next-hop virtual routers for OpenStack owned networks. This allows dynamic
routing changes to be communicated to the physical infrastructure based on
changes in OpenStack networks. Since OpenStack Networks are managed by users,
address scope limitations need to be in place.

A new charm is proposed called neutron-dynamic-routing which will configure and
deploy the neutron-bgp-dragent (Neutron BGP dynamic routing agent). The
neutron-bgp-dragent is a BGP speaker which can be peered with existing routers
in the BGP infrastructure. This charm will require interfaces for AMQP and
potentially neutron-api-plugin (TBD). For testing purposes it will also make
use of the bgp interface to peer with the quagga charm, both written by Frode
Nordahl (See Testing below.)

Requirements
------------

The charm will need to select an interface on the provider network for
communication with OpenStack. It may also need to select an interface to speak
with BGP routers in the infrastructure.

High availability is accomplished by deploying N number of units of
neutron-dynamic-routing with care for placement on different physical hosts.
This will introduce multiple neutron-bgp-dragents. BGP infrastructure routers
will need to be manually informed of each neutron-bgp-dragent.

IPv4 and IPv6 support is handled with separate bgp-speakers defined for each.

The charm will need to support both DVR and centralized OpenStack routing.

Outside the scope
-----------------

It is important to understand what the OpenStack dynamic routing implementation
does *not* do and lies outside of the scope of this spec.

* It does not provide ingress BGP to OpenStack, it is purely egress
  advertisement of OpenStack owned networks.
* It does not automatically advertise all OpenStack networks or whole subnet
  pools. A considerable amount of administrator configuration is still
  required, including associating OpenStack networks with the bgp-speaker. See
  the `Testing documentation https://docs.openstack.org/neutron-dynamic-routing/latest/contributor/testing.html>`__
  for a sense of the administrative overhead required.
* It does not provide a mechanism for injecting arbitrary BGP routes (for
  example for /32 FIPs). It merely advertises OpenStack networks that have been
  associated with the bgp speaker.

Alternatives
------------

Floating IPs, and NAT in general, are one approach to this problem, but there
are couple of drawbacks. For starters, NAT comes with a performance price as
number of connections grows. In cloud, where many VMs can establish connections
to various external peers, this can require significant memory and cpu
resources. Neutron gateway doesn't really scale without manual intervention. On
top of that, if only NAT is used, peers can’t really know which instance
established connection, they only know connection came ‘from the cloud’.
Same problems apply to DVR.

Static routes on physical gateways are the closest thing to BGP solution. The
only drawback is operational; in case address scope in Neutron changes, these
changes need to be implemented on physical routers too.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  thedac

Additional work:
  fnordahl

Gerrit Topic
------------

Use Gerrit topic "<topic_name>" for all patches related to this spec.

.. code-block:: bash

    git-review -t dragent

Work Items
----------

**New charm neutron-dynamic-routing**

A new reactive charm will be developed to configure and deploy the
neutron-bgp-dragent.

**Update neutron-api charm**

The neutron-api charm will need to load the service_plugin
neutron_dynamic_routing.services.bgp.bgp_plugin.BgpPlugin and subsequently run
a db migrate.

**Potential need for new neutron-api-plugin interface**

It is unclear at this time if a relationship to neutron-api will be required.
If it is required no layered interface for neutron-api-plugin yet exists and
will need to be created.

**New charm quagga**

For testing purposes the new quagga charm will act as a BGP peer to validate
and test neutron-dynamic-routing. Frode Nordahl has already begun work on this
project: https://github.com/openstack-charmers/charm-quagga

**New interface bgp**

For testing purposes the new bgp interface will define the peering relationship
between neutron-dynamic-routing and the quagga charm. Frode Nordahl has already
begun work on this project: https://github.com/openstack-charmers/interface-bgp


Repositories
------------

A new repository for the neutron-dynamic-routing charm is needed.

.. code-block:: bash

    git://git.openstack.org/openstack/charm-neutron-dynamic-routing

A new repository for the quagga charm is needed.

.. code-block:: bash

    git://git.openstack.org/openstack-charmers/charm-quagga

A new repository for the bgp interface is needed.

.. code-block:: bash

    git://git.openstack.org/openstack-charmers/interface-bgp


Documentation
-------------

An update to the charm deployment guide will be required.

Upstream documentation is found at the following:
https://docs.openstack.org/neutron-dynamic-routing/latest/

Security
--------

The neutron-dynamic-routing charm is dependent on the OpenStack implementation
of BGP. Which uses a simple string as a password.

Testing
-------

The neutron-dynamic-routing charm will need to be tested with a BGP peer. Frode
Nordahl has begun work on the quagga charm which will act as an infrastructure
BGP router allowing for testing and validation.

A mojo spec, or other suitably automated test, will be a requirement for this
feature implementation.


Dependencies
============

None
