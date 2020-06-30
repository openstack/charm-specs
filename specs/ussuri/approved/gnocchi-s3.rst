..
  Copyright 2020 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

============================================
Enablement of S3 storage backend for Gnocchi
============================================

Problem Description
===================

The gnocchi charm is a multi-tenant times eries, metrics and resources
database. It currently only supports connections to a Ceph storage
backend. For anyone using a S3 storage backend, this is an issue.

Proposed Change
===============

The proposed solution is to make the choice of storage backend a
configuration option. The relation between gnocchi and ceph-mon would become
an optional one, that would be activated only when Ceph is the selected backend.
The upstream documentation of Gnocchi mentions that S3 storage backends are
supported, along with others, such as Swift, Redis and Ceph
(https://gnocchi.xyz/install.html). The new feature would be supported on OpenStack
Stein (and later) on Ubuntu 18.04 LTS.


Alternatives
------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee: camille.rodriguez

Gerrit Topic
------------

Use Gerrit topic "s3-storage" for all patches related to this spec.

.. code-block:: bash

    git-review -t s3-storage

Work Items
----------

* Add a charm option for selecting S3

* Add options for S3 in the configuration file:

  * S3 endpoint URL
  * S3 region name
  * S3 access key id
  * S3 secret access key
  * Prefix to namespace metric bucket: s3_bucket_prefix = gnocchi
  * Maximum time to wait checking data consistency when writing to S3: s3_check_consistency_timeout = 60
  * The maximum number of connections to keep in a connection pool: s3_max_pool_connections = 50

* Make the Ceph relation and configuration optional, dependent on the charm option

* Implement https endpoint for S3

Timeline
-----------

The goal is to implement this change in the OpenstStack Charms 20.08 release. The freeze date for
this release is July 24th 2020, for a release on August 5th (see release
schedule). This change should be proposed for merging at least two weeks
ahead of freeze, so ideally submitted by July 10th 2020.

Repositories
------------

Changes will be pushed to the existing gnocchi charm repository:

https://git.openstack.org/openstack/charm-gnocchi


Documentation
-------------

* Update of the charm options

* Update of the charm README


Security
--------

No additional security concerns. The feature will be compatible
with an HTTPS endpoint to allow for encryption.

Testing
-------

Code changes will be covered by unit tests.
Functional tests would require S3 storage hardware or software emulation.
This is possible through a Swift deployment, or with RADOS Gateway. The deployment
of the storage backend with S3 capability would be independent from the
OpenStack bundle, to represent a real scenario.


Dependencies
============

No dependencies outside of this specification.
