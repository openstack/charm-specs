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

==========================
Ceph BlueStore Compression
==========================

The Ceph BlueStore storage backend has support for inline compression.

Problem Description
===================

The Ceph charms currently do not provide documentation, validation or support
for enabling BlueStore inline compression.

Proposed Change
===============

Ceph supports configuring compression through per-pool properties and global
configuration options. What configuration to use depends on the type of data
and the characteristics of the underlying storage devices in use.

To accomodate this we should allow Ceph consuming charms to configure
compression on a per-pool basis and have centrally managed defaults for any
options not provided.

The following configuration options will be added:

* bluestore-compression-algorithm

* bluestore-compression-mode

* bluestore-compression-required-ratio

* bluestore-compression-min-blob-size

* bluestore-compression-min-blob-size-hdd

* bluestore-compression-min-blob-size-ssd

* bluestore-compression-max-blob-size

* bluestore-compression-max-blob-size-hdd

* bluestore-compression-max-blob-size-ssd

Support will be provided for Ceph Mimic and onwards on Ubuntu 18.04 LTS and
onwards.

Alternatives
------------

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fnordahl

Gerrit Topic
------------

Use Gerrit topic "ceph-bluestore-compression" for all patches related to this
spec.

.. code-block:: bash

    git-review -t ceph-bluestore-compression

Work Items
----------

* Extend the Ceph broker API to support providing compression options on pool
  requests.

* Add context that can be used by both consuming and providing charms to map
  charm configuration options into CephBrokerRq op properties as well as Jinja2
  template context data.

* Extend the ``ceph-client`` and ``ceph-rbd-mirror`` reactive interfaces to
  support relaying compression options for pool creation.

* Add charm configuration options to control global compression defaults to the
  ceph-osd charm.

* Add charm configuration options to control compression on pool creation to
  the following charms:

  * glance

  * cinder-ceph

  * nova-compute

  * gnocchi

  * ceph-iscsi

  * ceph-radosgw

* Add support for copying compression properties on pool mirroring to the
  ceph-rbd-mirror charm.

* Establish a performance baseline and perform benchmark for use in a
  hyper-converged deployment topology and document results and recommendations.

Repositories
------------

No new repositories are required to implement this spec.

Documentation
-------------

The individual configuration options will get a description that explains basic
usage.

A new appendix or section should be added to the deployment-guide to document
results of benchmark and recommendations for deployment of Ceph with BlueStore
Compression.

Security
--------

Apart from enabling code paths in Ceph that was previously not exposed in the
charms, there are no security specific considerations for this feature work.

Testing
-------

Existing charm gates should be extended to validate the operation of the new
charm configuration options.

No special hardware is required to validate the feature.

Dependencies
============

None.
