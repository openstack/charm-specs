..
  Copyright 2016 Canonical Ltd

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=========================
Ceph Broker CephX Support
=========================

Problem Description
===================

Currently the ceph/ceph-mon charm provides cephx keys to clients which
have rw permissions to all pools in the ceph cluster; this is problematic
because it means that any client can read/write/delete data in any pool
so in the event of a compromise of a service (which might be directly
accessible by end users such as cinder, glance or the ceph-radosgw), the
entire cluster is compromised until the compromised key is revoked.

Proposed Change
===============

Ceph supports fine grained access control on keys, so we need to leverage
this functionality to improve the general security of a deployment, for
example (taken directly from the Ceph documentation):

.. code-block:: bash

    ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=vms, allow rx pool=images'
    ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images'
    ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups'

The ceph/ceph-mon will provide new broker methods to allow

a) Client requested pools to be placed into 'groups' at point of creation:

   For example, multiple cinder backend pools would be placed into the
   'volumes' group.

b) Clients can also request access to a group of pools:

   For example, the cinder charm will gain 'rwx' for pools in the 'volumes'
   group, and will request 'rx' permission for pools in the 'images' group.

c) Clients can optional request additional permissions:

   This supports the 'allow class-read object_prefix rbd_children' use-case
   where a key needs to be able to read from copy-on-write clones in different
   pools.

Alternatives
------------

Security could be left open and secured post deployment by the operator, but
this is neither repeatable or desirable from an operations perspective.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  xfactor973 (Chris Holcombe)


Gerrit Topic
------------

Use Gerrit topic "cephx-keys" for all patches related to this spec.

.. code-block:: bash

    git-review -t cephx-keys

Work Items
----------

- Implementation of group->pool mapping and maintenance in ceph-broker
- Implementation of cephx key->group ACL mapping in ceph-broker
- Implementation of methods on relation API to support mapping pools
  to groups.
- Implementation of methods on broker relation API to support granting
  access to pools to cephx keys with correct permissions.
- Implementation of cephx key update mechanism when membership of
  a pool group changes.
- Updates to cinder, glance, nova-compute and ceph-radosgw charms to
  make appropriate pool access requests to the ceph/ceph-mon charm.

Repositories
------------

No new repositories are required for this work.

Documentation
-------------

As the broker relation interface is self documenting, no additional
documentation is required.

Security
--------

This work mitigates an existing security risk.

Testing
-------

The existing charm functional tests will automatically cover this
work as they should be exercising the functionality of services
consuming ceph resources; incorrect key permissions will result in
broken function.

Unit tests should be written to validate the key and group management
functions in charms.ceph; additionally the ceph/ceph-mon charm
function tests should be updated to verify the permissions on created
keys in the current test cases.

Dependencies
============

- No external dependencies for this work.
