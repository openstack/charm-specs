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

==============
Barbican Charm
==============

Provide a charm for deploying Barbican with support for associated
HSM modules/devices.

Problem Description
===================

OpenStack services and users often need a repository to store sensitive
information like passwords, cryptographic keys etc. The Barbican service
provides an interface on top of an HSM for doing that.

Proposed Change
===============

One new charm - Barbican;  Charm needs to take into account potential use of
backed hardware security modules (HSM) - this might be nicely done using the
cinder-backend approach as a subordinate charm to avoid polluting the main
charm with details of every HSM possible.

The new charm, as a minimum, should include the following features:

- Deployable in a highly available configuration
- Allow clients and services to interact using SSL encryption
- Charm progress displayed via workload status

Alternatives
------------

Secrets stored via other means outside of OpenStack.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ajkavanagh
  gnuoy

Gerrit Topic
------------

Use Gerrit topic "barbican" for all patches related to this spec.

.. code-block:: bash

    git-review -t barbican

Work Items
----------

Provide base and interface layers required for OpenStack charms
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

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

Provide Barbican charm
++++++++++++++++++++++

- Create skeleton charm layer based on OpenStack base layer and available
  interface layers to deploy Barbican.
- Add support for upgrading Barbican
- Add config option and accompanying support to enable barbicans use of
  configurable storage backends: ie. HSM (hardware security module)
  NOTE: configuration without HSM is not secure and is for testing purposes
  only.
- Add config option and accompanying support for upgrades via
  action-managed-upgrade.
- Add support for deploying Barbican in a highly available configuration
- Add support for the Barbican to display workload status
- Add support SSL endpoints
- Charm should have unit and functional tests.

Mojo specification deploying and testing Barbican
+++++++++++++++++++++++++++++++++++++++++++++++++

- Write Mojo spec for deploying Mojo in an HA configuration and testing
  storage and retrieval of secrets.

Repositories
------------

A new git repository will be required for the Barbican charm:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-barbican

Documentation
-------------

The Barbican charm should contain a README with instructions on deploying the
charm. A blog post is optional but would be a useful addition.

Security
--------

Given the purpose of Barbican is to store and manage secrets a review of the
charm by the security team may be appropriate.

Testing
-------

Code changes will be covered by unit tests; functional testing will be done
using a combination of Amulet, Bundle tester and Mojo specification.

Dependencies
============

None
