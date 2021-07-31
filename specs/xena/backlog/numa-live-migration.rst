..
  Copyright 2019 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

==========================
SR-IOV Port Live Migration
==========================

A recent change in Nova for Train release introduced the ability to
live migrate instances with NUMA topology (CPU Pinning and Huge Pages
also considered). This spec is to discuss and ensure support of that
feature in a charm deployment.

Problem Description
===================

As an operator, I would like the ability to live migrate instances
with NUMA topology.

Proposed Change
===============

In Nova several features depend of the internal NUMA topology
object. As indicated in the problem description an operator may ask
the desir of migrating such instances.

- NUMA awareness when flavor is configured with `hw:numa_nodes=N`
- CPU Pinning, when flavor is confiugred with `hw:cpu_policy=dedicated`
- Huge Pages, when flavor is configured with `hw:mem_page_size=large`

The support of this improvement will be considered as tech-preview for
Ubuntu 19.10/Train and production-ready for Ubuntu 18.04 LTS/Train and
Ubuntu 20.04 LTS/Ussuri.

No changes are expected to any of the charms. The aim here is to
validate behavior and provide additional documentations.

Alternatives
------------

An operator could identify a destination host with the same
characteristics as the source host and ensure that the host CPUs where
the guest vCPUs are pinned on are free. Also, in case of huge pages
usage, ensure the availability of the huge pages in the destination
host.

By default the availibity of live migrate instances with NUMA topology
is disabled in Nova but operators can enable that behavior using:

  ``workaround/enable_numa_live_migration=True``

It is to note that this option is not currently exposed in
``charm-nova-compute``.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Sahid Orentino Ferdjaoui <lp:sahid-ferdjaoui>

Gerrit Topic
------------

Use Gerrit topic "numa-live-migration" for all patches related to this spec.

.. code-block:: bash

    git-review -t numa-live-migration

Work Items
----------

The implementation consists of the following items:

- Validate behavior using NUMA awarness.
- Validate behavior using CPU Pinning.
- Validate behavior using Huge Pages.

Repositories
------------

- Any additional documentation will be considered for `charm-deployment-guide`.

Documentation
-------------

n/a

Security
--------

n/a

Testing
-------

n/a

Dependencies
============

- See support of NUMA aware live migration spec [0].

[0] https://specs.openstack.org/openstack/nova-specs/specs/stein/approved/numa-aware-live-migration.html
