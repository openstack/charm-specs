..
  Copyright 2018 Aakash KT

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
OpenStack with OVN
===============================

Openstack can be deployed with a number of SDN solutions (e.g. ODL). OVN
provides virtual-networking for Open vSwitch (OVS). OVN has a lot of desirable
features and is designed to be integrated into Openstack, among others.

Since there is already a networking-ovn project under openstack, it is the
obvious next step to implement a Juju charm that provides this service.

Problem Description
===================

Currently, Juju charms have support for deploying openstack, either with it's
default SDN solution (Neutron), or with others such as ODL. This project
will expand the deployment scenarios under Juju for openstack by including OVN
in the list of available SDN solutions.

This will also benefit OPNFV's JOID installer in providing another scenario in
its deployment.

Proposed Change
===============

Charms implementing neutron-api, ovn-controller and neutron-ovn will need to
be implemented. These will be written using the new reactive framework
of Juju.

Charm : neutron-ovn
-------------------

This charm will be deployed alongside nova-compute deployments. This will be
a subordinate charm to nova-compute, that installs and runs openvswitch and
the ovn-controller.

Charm : ovn-controller
----------------------

This charm will deploy ovn itself. It will start the OVN services
(ovsdb-server, ovn-northd). Since there can only be a single instance of
ovsdb-server and ovn-northd in a deployment, we can also implement passive
HA, but this can be included in further revisions of this charm.

Charm : neutron-api-ovn
-----------------------

This charm will provide the api only integration of neutron to OVN. This charm
will need to be subordinate to the existing neutron-api charm. The main task
of this charm is to setup the "neutron.conf" and "ml2_ini.conf" config files
with the right parameters for OVN. The principal charm, neutron-api, handles
the install and restart for neutron-server.

Refer for more information : https://docs.openstack.org/networking-ovn/latest/install/manual.html

Alternatives
------------

N/A

Implementation
==============

Assignee(s)
-----------
Aakash KT

Gerrit Topic
------------

Use Gerrit topic "charm-os-ovn" for all patches related to this spec.

.. code-block:: bash

    git-review -t charm-os-ovn

Work Items
----------

* Implement neutron-api-ovn charm
* Implement ovn-controller charm
* Implement neutron-ovn charm
* Integration testing
* Create a bundle to deploy OpenStack OVN
* Create documentation for above three charms

Repositories
------------

Yes, three new repositories will need to be created :
* charm-neutron-api-ovn
* charm-ovn-controller
* charm-neuton-ovn

Documentation
-------------

This will require creation of new documentation for the covered scenario of
openstack + ovn in Juju.
A README file for the bundle needs to be written.
Add documentation in charm-deployment guide to detail how to deploy OVN with
OpenStack.

Security
--------

Communications to and from these charms should be made secure. For eg.
communication between ovn-central and ovn-edges to be made secure using
self-signed certs.

Testing
-------

For testing at Juju level, we can use the new "juju-matrix" tool.
For testing functionality at OpenStack level, Mojo should be used. This will
help validate the deployment.

Dependencies
============

This charm will support OpenStack Queens as its baseline.
