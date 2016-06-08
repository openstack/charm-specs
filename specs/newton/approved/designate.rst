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

===============
Designate Charm
===============

Charm OpenStack DNS (Designate).

Problem Description
===================

Designate (DNSaaS) provides a method for managing DNS records for addressing
OpenStack guests and floating IPs.

Proposed Change
===============

Designate will need to undergo MIR review for main inclusion; this process
should include stripping of all debconf/dbconfig related code from the
packaging.

Two new charms - designate and designate-bind - designate provides the API/RPC
services, and bind provides a flexible scale out service for managing BIND DNS
servers; the interfaces between the two charms should be sufficiently flexible
to allow different DNS backends to be plugged in at a later date if require
(i.e. a PowerDNS charm instead of BIND).

The new Designate charm should include, as a minimum, the following features:

- Deployable in a highly available configuration
- Allow clients and services to interact using SSL encryption
- Charm progress displayed via workload status

Alternatives
------------

DNS records could be managed manually in an existing DNS server.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gnuoy

Gerrit Topic
------------

Use Gerrit topic "designate" for all patches related to this spec.

.. code-block:: bash

    git-review -t designate

Work Items
----------

Provide fully supported packages for Ubuntu
+++++++++++++++++++++++++++++++++++++++++++

- Package updates for Designate to strip all debconf/dbconfig related code from
  the packaging.
- MIR review for Designate - Evaluate package and source according to
  https://wiki.ubuntu.com/MainInclusionProcess, open corresponding bug, work
  with Ubuntu MIR team and make any other necessary package changes to get
  package into main.

Provide Designate charm
+++++++++++++++++++++++

- Create skeleton charm layer based on OpenStack base layer and available
  interface layers to deploy Designate.
- Add support for upgrading Designate
- Add config option and accompanying support for upgrades via
  action-managed-upgrade.
- Add support for deploying Designate in a highly available configuration
- Add support for the Designate to display workload status
- Add support SSL endpoints
- Charm should have unit and functional tests.

Provide Designate Bind charm
++++++++++++++++++++++++++++

- Create bind charm designed to integrate with designate.
- Ensure charm meets basic non-functional requirements, such as HA and workload
  status

Extend Designate charm
++++++++++++++++++++++

- Add support for Designate integration with Neutron to extend the
  automatically created record information.

Mojo specification deploying and testing Designate
++++++++++++++++++++++++++++++++++++++++++++++++++

- Write Mojo spec for deploying Mojo in an HA configuration and testing
  automatic and manual creation of DNS records.

Repositories
------------

New git repositories will be required for the Designate and
Designate Bind charms:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-designate
    git://git.openstack.org/openstack/charm-designate-bind

Documentation
-------------

The Designate charm should contain a README with instructions on deploying the
charm. A blog post is optional but would be a useful addition.

Security
--------

No additional security concerns.

Testing
-------

Code changes will be covered by unit tests; functional testing will be done
using a combination of Amulet, Bundle tester and Mojo specification.

Dependencies
============

- Provide rabbitmq interface layer
- Provide mysql-shared interface layer
- Provide pgsql interface layer
- Provide keystone interface layer
- Provide hacluster interface layer
- Provide nrpe-external-master interface layer
- Provide OpenStack base layer with all common hook code that is not already
  covered by an interface layer.
- Provide OpenStack base layer with support for HA deployments
- Provide OpenStack base layer with support for SSL communication
- Provide OpenStack base layer with support for workload status
