..
  Copyright 2017, Canonical UK

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=============
Gnocchi Charm
=============

Storage of metrics in an OpenStack Cloud is a big data problem.

Ceilometer has used MongoDB since its initiation for storage of measures;
however this has not proven scalable, and the telemetry project have
moved to a default of using Gnocchi (a spin out of the telemetry project)
for storage of measures.

We need to charm Gnocchi to allow OpenStack Operators to deploy and use
telemetry services as part of an OpenStack cloud.

Problem Description
===================

Gnocchi is a multi-tenant timeseries, metrics and resources database.
It provides an HTTP REST interface to create and manipulate the data.
It is designed to store metrics at a very large scale while providing
access to metrics and resources information and history.

Proposed Change
===============

The new Gnocchi charm should include, as a minimum, the following features:

- Deployable in a highly available configuration
- Allow clients and services to interact using SSL encryption
- Charm progress displayed via workload status

The charm will use the default storage options:

- Storage driver: Ceph (preferred)
- Indexing driver: MySQL (not preferred but default for rest of OpenStack)
- Coordination: Memcache (zookeeper and redis are alternative options)

The choice of coordination driver is limited to memcache, zookeeper or redis
as these drivers support the required features for Gnocchi.

The Gnocchi charm will support OpenStack Mitaka and later on Ubuntu 16.04.

Graphical reporting for Ceilometer was dropped from Horizon during Ocata;
the telemetry stack will make use of the Grafana Charm + the Gnocchi Plugin
for graphical reporting on Gnocchi metrics.

Alternatives
------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jamespage

Gerrit Topic
------------

Use Gerrit topic "gnocchi-charm" for all patches related to this spec.

.. code-block:: bash

    git-review -t gnocchi-charm

Work Items
----------

Reactive Interfaces
+++++++++++++++++++

- interface: gnocchi

Provide Gnocchi charm
+++++++++++++++++++++

- Create skeleton charm layer based on OpenStack base layer and available
  interface layers to deploy Gnocchi.
- Add support for upgrading Gnocchi
- Add config option and accompanying support for upgrades via
  action-managed-upgrade.
- Add support for deploying Gnocchi in a highly available configuration
- Add support for the Gnocchi to display workload status
- Add support SSL endpoints
- Charm should have unit and functional tests.

Update Ceilometer and Aodh charms
+++++++++++++++++++++++++++++++++

- Support for deployment with Gnocchi

Mojo specification deploying and testing Gnocchi
++++++++++++++++++++++++++++++++++++++++++++++++

- Update HA Mojo spec for deploying Gnocchi in an HA configuration.

Update telemetry bundle
+++++++++++++++++++++++

- Update telemetry bundle to deploy Gnocchi and Grafana.

Repositories
------------

A new git repository will be required for the Gnocchi charm:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-gnocchi

Documentation
-------------

The Gnocchi charm should contain a README with instructions on deploying the
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

- No dependencies outside of this specification.
