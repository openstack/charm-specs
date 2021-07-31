..
  Copyright 2021 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

====================================================
OpenStack Compute Nodes Juju Availability Zones Sync
====================================================

For some providers, Juju may set the environment variable
``JUJU_AVAILABILITY_ZONE`` with information about the availability zone of the
machine deployed.

OpenStack has the concept of availability zones too, and it would be useful
for Juju operators to have the option of automatically sync the Juju
availability zones with the OpenStack availability zones.

Problem Description
===================

At the moment, a Juju operator needs to manually sync the nova-compute Juju
machines' availability zones (AZs), with the OpenStack AZs.

Proposed Change
===============

A new Juju action to the nova-cloud-controller charm will be implemented to
automatically sync the Juju AZs with the OpenStack AZs. The availability zone
for each Juju unit is exposed to the nova-cloud-controller charm through the
cloud-compute relation.

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ionutbalutoiu

Gerrit Topic
------------

Use Gerrit topic "compute-nodes-az-sync" for all patches related to this spec.

.. code-block:: bash

    git-review -t compute-nodes-az-sync

Work Items
----------

- nova-compute

  - Set a new relation variable called ``availability_zone`` in the
    ``cloud-compute`` relation (used to relate to the ``nova-cloud-controller``
    application). This variable will take the value of the automatic Juju
    variable ``JUJU_AVAILABILITY_ZONE``, if this is set by the underlying
    provider.

- nova-cloud-controller

  - Implement a new Juju action called ``sync-compute-availability-zones`` used
    to sync the Juju availability zones, set by the remote units in the
    ``cloud-compute`` relation, with the OpenStack availability zones.

Repositories
------------

- openstack/charm-nova-compute

- openstack/charm-nova-cloud-controller

Documentation
-------------

The new Juju action needs to be documented in the appropriate docs that
reference the nova-cloud-controller actions. This is the only user-facing
change.

Security
--------

None

Testing
-------

Code written or changed will be covered by unit tests.

Dependencies
============

The ``nova-cloud-controller`` charm does not install the Nova client Python
library. This is needed for the new Juju action, so it is added to the charm
Python dependencies.
