..
  Copyright 2020, Canonical Ltd.

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
Ceph Erasure Coded Pool Support
===============================

Problem Description
===================

Ceph Erasure Coded pools provide an alternative to Replicated pools with
comparable performance whilst requiring less raw block device storage for
the same effective usable capacity.

The Ceph charms and charms that consume ceph services don't currently
provide support for use of Erasure Coded pools.

Proposed Change
===============

The ceph-{mon,proxy} broker will be extended to support the creation of
erasure-coded pools.  Some code exists in the broker to support EC pools
however it has never been tested so may require some updates and refactoring.

Some existing support exists in both charmhelpers and charms.ceph to support
the creation of erasure coded pools; however it is largely untested and will
need to be validated and updated as part of the work to enable support
for Erasure Coding in the OpenStack Charms.  Existing gaps include support
for EC overwrites (required for most RBD use-cases) and providing the
minimum size for the EC pool.

Erasure Coding has been supported by Ceph for some time however overwrites
(required for RBD uses cases) where not implemented until Luminous so the
minimum version requirement for Ceph is Luminous as shipped with Ubuntu
18.04 LTS.

Ceph deployments must be using Bluestore - Filestore based deployments
cannot support RBD use-cases without the use of a cache tier (which is
not supported in charmed deployments).

Consuming charms will be updated to include new configuration options to
support the use of EC pools - this includes the following charms:

- glance (rbd)
- cinder-ceph (rbd)
- nova-compute (rbd)
- ceph-radosgw
- ceph-iscsi (rbd)
- ceph-fs

.. list-table:: Erasure Coding charm options
   :widths: 25 5 15 55
   :header-rows: 1

   * - Option
     - Type
     - Default Value
     - Description
   * - pool-type
     - string
     - replicated
     - Ceph pool type to use for storage - valid values are 'replicated' and 'erasure-coded'.
   * - ec-profile-name
     - string
     -
     - Name for the EC profile to be created for the EC pools. If not defined a profile name will be generated based on the name of the pool used by the application.
   * - ec-rbd-metadata-pool
     - string
     -
     - Name of the metadata pool to be created (for RBD use-cases).  If not defined a metadata pool name will be generated based on the name of the data pool used by the application.  The metadata pool is always replicated (not erasure coded).
   * - ec-profile-k
     - int
     - 1
     - Number of data chunks that will be used for EC data pool. K+M factors should never be greater than the number of available AZs for balancing.
   * - ec-profile-m
     - int
     - 2
     - Number of coding chunks that will be used for EC data pool. K+M factors should never be greater than number of available AZs for balancing.
   * - ec-profile-locality
     - int
     -
     - (lrc plugin - l) Group the coding and data chunks into sets of size l. For instance, for k=4 and m=2, when l=3 two groups of three are created. Each set can be recovered without reading chunks from another set.  Note that using the lrc plugin does incur more raw storage usage than isa or jerasure in order to reduce the cost of recovery operations.
   * - ec-profile-crush-locality
     - string
     -
     - (lrc plugin) The type of the crush bucket in which each set of chunks defined by l will be stored. For instance, if it is set to rack, each group of l chunks will be placed in a different rack. It is used to create a CRUSH rule step such as 'step choose rack'. If it is not set, no such grouping is done.
   * - ec-profile-durability-estimator
     - int
     -
     - (shec plugin - c) The number of parity chunks each of which includes each data chunk in its calculation range. The number is used as a durability estimator. For instance, if c=2, 2 OSDs can be down without losing data.
   * - ec-profile-helper-chunks
     - int
     -
     - (clay plugin - d) Number of OSDs requested to send data during recovery of a single chunk. d needs to be chosen such that k+1 <= d <= k+m-1. The larger the d, the better the savings.
   * - ec-profile-scalar-mds
     - string
     -
     - (clay plugin) Specifies the plugin that is used as a building block in the layered construction. It can be one of: jerasure, isa or shec.
   * - ec-profile-plugin
     - string
     - jerasure
     - EC plugin to use for this applications pool. These plugins are available: jerasure, lrc, isa, shec, clay.
   * - ec-profile-technique
     - string
     - reed_sol_van
     - EC profile technique used for this applications pool - will be validated based on the plugin configured via ec-profile-plugin. Supported techniques are 'reed_sol_van', 'reed_sol_r6_op', 'cauchy_orig', 'cauchy_good', 'liber8tion' for jerasure, 'reed_sol_van', 'cauchy' for isa and 'single', 'multiple' for shec.
   * - ec-profile-device-class
     - string
     -
     - Device class from CRUSH map to use for placement groups for erasure profile - valid values: ssd, hdd or nvme (or leave unset to not use a device class).

RBD use cases require a metadata pool which is always configured as
a replicated pool in addition to the underlying EC coded pool for
volume data storage.  The consuming application is configured to use
the metadata pool, with the data pool being configured via
'ceph.conf' for the specific application - for example:

.. code-block:: none

   rbd default data pool = glance

As OpenStack services specify a full path to an application specific
ceph.conf configuration file, the default data pool configuration
does not need to be scoped to a specific client identifier.

The ceph-mon interface implementations for both reactive and operator
framework charms will be updated to include support for EC.

OpenStack version support will be validated as part of the implementation
of this specification.

This feature is not dependent on a specific version of Juju.

Alternatives
------------

EC pools can be created post deployment using actions provided by the ceph-mon
charm. However other than forking the consuming charms, no way currently exists
to provide the required configuration to Ceph and OpenStack services to consume
EC pools in RBD use-cases.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

- james-page
- gnuoy

Gerrit Topic
------------

Use Gerrit topic "ceph-erasure-coding" for all patches related to this spec.

.. code-block:: none

   git-review -t ceph-erasure-coding

Work Items
----------

Ceph core EC enablement:

- charms.ceph: Review and add required features for RBD EC pool usage.
- charmhelpers: Review and add required features for RBD EC pool usage.
- ceph-mon: Resync updates to charms.ceph + charmhelpers and update
  the broker code base to support any required changes.
- ceph-proxy: Resync updates to charms.ceph + charmhelpers and update
  the broker code base to support any required changes.

Interfaces:

- charm-interface-ceph-client: Update to add support for EC pool and
  profile creation.
- ops-interface-ceph-client: Update to add support for EC pool and
  profile creation.
- charm-interface-ceph-mds: Update to add support for EC pool and
  profile creation.

- charm-interface-ceph-rbd-mirror: Review for any EC pool requirements
  based on support for EC in ceph rbd-mirror.

OpenStack RBD use cases:

- glance: Add required configuration option support and updates to the
  ceph broker interaction to support EC pools. Add functional testing
  to cover at least earliest and latest supported releases as part of a
  recheck-full pre-commit test run.
- cinder-ceph: Add required configuration option support and updates to the
  ceph broker interaction to support EC pools. Add functional testing
  to cover at least earliest and latest supported releases as part of a
  recheck-full pre-commit test run.
- nova-compute: Add required configuration option support and updates to the
  ceph broker interaction to support EC pools. Add functional testing
  to cover at least earliest and latest supported releases as part of a
  recheck-full pre-commit test run.

Ceph RBD use cases:

- ceph-iscsi: Add required configuration option support and updates to the
  ceph broker interaction to support EC pools. Add functional testing
  to cover at least earliest and latest supported releases as part of a
  recheck-full pre-commit test run.
- ceph-rbd-mirror: Validate EC pools are not supported with rbd-mirror,
  document in release notes and charm deployment guide.

Other use cases:

- ceph-radosgw: Add required configuration option support and updates to the
  ceph broker interaction to support EC pools. Add functional testing
  to cover at least earliest and latest supported releases as part of a
  recheck-full pre-commit test run.  All pools aside from the .data pool
  must continue to be replicated pools.

- ceph-radosgw: Validate support for multi-site RADOS gateway replication.

- ceph-fs: Add required configuration option support and updates to the
  ceph broker interaction to support EC pools. Add functional testing
  to cover at least earliest and latest supported releases as part of a
  recheck-full pre-commit test run.

Repositories
------------

No new git repositories are required.

Documentation
-------------

The charm deployment guide will be updated to detail use of the OpenStack
Charms with Ceph Erasure Coded pools.  This will include details on how
to structure zones within a Ceph deployment using EC pools as the requirements
are potentially quite different to replicated pools.

Security
--------

No additional security attack surface is exposed by use of this feature.

Testing
-------

No special hardware is required for testing of EC pools.

Functional tests will be added to consuming charms to validate the creation
and use of EC pools across OpenStack and Ceph use-cases.

Performance testing of EC pools will be undertaken as part of this
spec to benchmark EC pool configurations against replicated pool
configurations.

Dependencies
============

No dependencies on other specs.
