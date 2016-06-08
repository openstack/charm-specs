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

===============================
CephFS
===============================

CephFS has recently gone GA and we can now move forward with offering
this to people as a charm. Up until this point it was considered too
experimental to store production data on.

Problem Description
===================

A new CephFS charm will be created. This charm will leverage the Ceph
base layer.

Proposed Change
===============

A new CephFS charm will be created. This charm will leverage the Ceph
base layer. The effort required should be small once the Ceph base
layer is ready to go.

Alternatives
------------

GlusterFS is an alternative to CephFS and will probably fit many users
needs. However there are users who don't want to deploy additional
hardware to create another cluster so this can be convenient in those
cases.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  cholcombe973


Gerrit Topic
------------

Use Gerrit topic "cephfs" for all patches related to this spec.

.. code-block:: bash

    git-review -t cephfs

Work Items
----------

ceph-fs charm
+++++++++++++

- Create the ceph-fs charm utilizing the base Ceph layer
- Expose interesting config items such as: mds cache size, mds bal mode
- Create actions to allow blacklisting of misbehaving clients, breaking
  of locks, creating new filesystems, add_data_pool, remove_data_pool,
  set quotas, etc.

cephfs-interface
++++++++++++++++

- Create an interface to allow other charms to mount the filesytem.

Repositories
------------

A new git repository will be required to host the ceph-fs charm:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-ceph-fs

Documentation
-------------

A README.md will be created for the charm as part of the normal development
workflow.

Security
--------

No additional security concerns.

Testing
-------

A mojo spec will be developed to exercise this charm along with amulet tests
if needed:

 * Deploy ceph-mon
 * Deploy ceph-osd
 * Deploy cephfs
 * Relate the three
 * Verify that CephFS can be mounted and responds to reads/writes

Dependencies
============

- This project depends on the Ceph Layering project being successful.
