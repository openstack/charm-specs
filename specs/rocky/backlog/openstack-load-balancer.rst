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

================================
OpenStack Endpoint Load Balancer
================================

To enable Openstack services for a single cloud to be installed in a highly
available configuration without requiring that each unit of a service is in
the same broadcast domain.

Problem Description
===================

1. As a cloud administrator I would like to simplify my deployment so that I
   don't have to manage a corosync and pacemaker per OpenStack API service.

2. As a cloud architect I am designing a new cloud where all services will be
   in a single broadcast domain. I see no need to use the new central
   loadbalancer and would like to continue to have each service manage its
   own VIP.

3. As a cloud architect I would like to spread my control plane across N racks
   for redundancy. Each rack is in its own broadcast domain. I do not want the
   users of the cloud to require knowledge of this topology. I want the
   endpoints registered in Keystone to work regardless of a rack level failure.
   I am using network spaces to segregate traffic in my cloud and the OpenStack
   loadbalancer has access to all spaces so I only require one set of
   loadbalancers for the deployment.

4. As a cloud architect I would like to spread my control plane across N racks
   for redundancy. Each rack is in its own broadcast domain. I do not want the
   users of the cloud to require knowledge of this topology. I want the
   endpoints registered in Keystone to work regardless of a rack level failure.
   I am using network spaces to segregate traffic in my cloud. I want the
   segregation to extend to the load balancers and so will be requiring a set
   of load balancers per network space.

5. As a cloud architect I am designing a new internal cloud and have no
   interest in IPv6, I wish to deploy a pure IPv4 solution.

6. As a cloud architect I am designing a new cloud. I appreciate that it has
   been 18 years since the IETF brought us IPv6 and feel it maybe time to
   enable IPv6 within the cloud. I am happy to have some IPv4 where needed
   and am looking to deploy a dual stack IPv4 and IPv6.

7. As a cloud architect I am designing a new cloud. I appreciate that it has
   been 18 years since the IETF brought us IPv6 and wish to never see an IPv4
   address again. I am looking to deploy a pure IPv6 cloud.

8. As a cloud architect I wish to use DNS HA in conjunction with the OpenStack
   loadbalancer so that loadbalancer units can be spread across different
   subnets within each network space.

9. As a cloud administrator I would like to have the OpenStack load balancers
   look after HA and so will be deploying in an Active/Passive deployment.
   I will need to use a VIP for the loadbalancer in this configuration.

10. As a cloud architect I have an existing hardware loadbalancers I wish to
    use. I do not want to have to update it with the location of each API
    service backend. Instead I would like to have the OpenStack load balancers
    in an Active/Active configuration and have the hardware loadbalancers
    manager traffic between haproxy instance in the OpenStack loadbalancer
    service. I do not need to use a VIP for the loadbalancer in this
    configuration. My hardware loadbalancers utilise vip(s) which will need
    to be registered as the endpoints for services in Keystone.

11. As a cloud administrator haproxy statistics are fascinating to me and I
    want the statistics from all haproxy instances to be aggregated.

12. As a cloud administrator I would like haproxy to be able to perform health
    checks on the backends which assert the health of a service more
    conclusively than simple open port checking.

13. As a cloud administrator I want to be able to configure max connections
    and timeouts as my cloud evolves.

14. As a charm author of a service which is behind the OpenStack load balancer
    I would like the ability to tell the loadbalancer to drain connection to a
    specific unit and take it out of service. This will allow the unit to go
    into maintenance mode.

Proposed Change
===============

New interface: openstack-api-endpoints
--------------------------------------

This interface allows a backend charm hosting API endpoints to inform
the OpenStack loadbalancer which services it's hosting and on which IP
address and port frontend API requests should be sent to on the backend
unit.  It also allows the backend charm to inform the loadbalancer which
frontend port should be used for each service.

Example - neutron-api (single API endpoint per unit):

.. code-block:: yaml

    endpoints:
      - service-type: network
        frontend-port: 9696
        backend-port: 9689
        backend-ip: 10.10.10.1
        check-type: http

Example - nova-cloud-controller (multiple API endpoints per unit):

.. code-block:: yaml

    endpoints:
      - service-type: nova
        frontend-port: 8774
        backend-port: 8764
        backend-ip: 10.10.10.2
        check-type: http
      - service-type: nova-placement
        frontend-port: 8778
        backend-port: 8768
        backend-ip: 10.10.10.2
        check-type: http

A single instance of the OpenStack Loadbalancer application will only service
a single type of OpenStack API endpoint (public, admin or internal).  The
charm will use the network space binding of the frontend interface to determine
which IP or VIP (if deployed in HA configuration) should be used by the
backend API service for registration into the Cloud endpoint catalog.

Having processed the requests from all backend units, the loadbalancer now
needs to tell the backend application the external IP being used to listen for
connections for each endpoint service type:

.. code-block:: yaml

    endpoints:
      - service-type: nova
        frontend-ip: 98.34.12.1
        frontend-port: 8774
      - service-type: nova-placement
        frontend-ip: 98.34.12.1
        frontend-port: 8778

The backend service now updates the endpoints in the Keystone registry to point
at the IPs passed back by the loadbalancer.

This interface is provided by each backend API charm and consumed via
the backend interface on the OpenStack loadbalancer charm.  Each backend
charm would provide three instances of this interface type:

.. code-block:: yaml

    provides:
      public-backend:
        interface: openstack-api-endpoints
      admin-backend:
        interface: openstack-api-endpoints
      internal-backend:
        interface: openstack-api-endpoints

Taking this approach means that the backend charm can continue to be the
entry point/loadbalancer for some endpoint types, and push the loadbalancing
for other entry points out to the OpenStack Loadbalancer charm (or multiple
instances).

Updates to keystone endpoint calculation code
---------------------------------------------

Currently the following competing options are used to calculate which EP should
be registered in Keystone:

* os-\*-network set do resolve_address old method
* dnsha use dnsha
* os-\*-hostname set use hostname
* juju network space binding via extra-bindings
* prefer ipv6 via configuration option
* presence of {public,internal,admin}-backend relations to
  opentack loadbalancers

OpenStack Loadbalancer charm
----------------------------

New charm - OpenStack Loadbalancer - with corresponding tests & QA CI/setup.

Alternatives
------------

1. Extend existing HAProxy charm.
2. Use DNS HA.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  unknown

Gerrit Topic
------------

Use Gerrit topic "osbalancer" for all patches related to this spec.

.. code-block:: bash

    git-review -t osbalancer

Work Items
----------

Provide OpenStack Loadbalancer Charm
++++++++++++++++++++++++++++++++++++

- Write draft interface for LB <-> Backend
- Write unit tests for Keystone endpoint registration code
- Write Keystone endpoint registration code


Mojo specification deploying and testing Mistral
++++++++++++++++++++++++++++++++++++++++++++++++

- Write Mojo spec for deploying LB in an HA configuration

Repositories
------------

A new git repository will be required for the Mistral charm:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-openstack-loadbalancer

Documentation
-------------

The OpenStack Loadbalancer charm should contain a README with instructions on
deploying the charm. A blog post is optional but would be a useful addition.

Security
--------

No additional security concerns.

Testing
-------

Code changes will be covered by unit tests; functional testing will be done
using a combination of Amulet, Bundle tester and Mojo specification.

Dependencies
============

None
