..
  Copyright 2019 Canonical

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

======================
CephFS NFS with Manila
======================

CephFS should be used to provide shared storage to OpenStack servers
via Manila.

Problem Description
===================

The existing CephFS charm doesn't have any usable integration with
Manila.

Proposed Change
===============

To add support for CephFS as a shared filesystem for an OpenStack Charm
deployed OpenStack, a few things are required. Primarily, a new charm will
be needed to bridge the NFS to CephFS protocols. The best option for this
application is Ganesha. In addition to charming Ganesha, additional work
will need to be done to improve the testability and supportablity of the
existing CephFS charm.

To allow the new Ganesha charm to integrate with Manila, a new
remote-manila-plugin interface will be added to the existing Manila charm
that will act exactly the same as the existing manila-plugin interface, but
will not be container scoped. This remote-manila-plugin interface will be used
to disable the manila-share service on the Manila charm. The new manila-ganesha
charm will configure the manila-share service in addition to Ganesha.

Access to the Ganesha network will include security groups restricting tenants
on the provider network from being able to access anything over that network
excepting their own NFS share, and IP restrictions will be used on the NFS
shares to restrict tenants from being able to access each others' shares.
This provider network will be documented as requiring creation by the OpenStack
administrator. The network will need to be created to match the space that the
Ganesha charm provides its NFS service over.

Alternatives
------------

As an alternative to installing Ganesha as the NFS gateway, it is possible to
implement a shared filesystem via CephFS and Manila directly. This approach
is not preferred because of the requirement to add the ceph-fs clients to all
tenants that would like to access the shared filesystems. In addition, letting
clients connect directly to Ceph reqiures exposing the Ceph cluster (mons and
OSDs) directly to tenant workloads, dramatically increasing the security
exposure to the shared storage.

Implementation
==============

A bundle overlay to add Ceph backed storage to Manila with Ganesha would look
like:

.. code-block:: yaml

    applications:
      manila-ganesha:
        charm: manila-ganesha
        num_units: 2
    relations:
      - - ceph-mon
        - ganesha
      - - manila-ganesha:remote-manila-plugin
        - manila:remote-manila-plugin

A more complete bundle could look like:

.. code-block:: yaml

    applications:
      ceph-mon:
        charm: cs:~openstack-charmers-next/ceph-mon
        num_units: 3
      ceph-osd:
        charm: cs:~openstack-charmers-next/ceph-osd
        num_units: 3
        storage:
          osd-devices: '10g'
      ceph-mds:
        charm: cs:~openstack-charmers-next/ceph-mds
        num_units: 2
      manila-ganesha:
        charm: manila-ganesha
        num_units: 2
      manila:
        charm: cs:~openstack-charmers-next/manila
        num_units: 1
    relations:
      - - ceph-mon
        - manila-ganesha
      - - ceph-mon
        - ceph-osd
      - - manila-ganesha:remote-manila-plugin
        - manila:remote-manila-plugin

Assignee(s)
-----------

Primary assignee:
  Chris.MacNaughton

Gerrit Topic
------------

Use Gerrit topic "charm-cephfs-manila" for all patches related to this spec.

.. code-block:: bash

    git-review -t charm-cephfs-manila

Work Items
----------

- New Manila-Ganesha charm
- Updates to the Manila charm
- Improvements to CephFS charm

Repositories
------------

There will be a few new repositories needed:

- charm-manila-ganesha

In addition, charm-manila and charm-ceph-fs charm will be updated.

Documentation
-------------

This change will require new documentation to the charm-guide, in addition
to new documentation in the new charms. Additionally, charm-ceph-fs will
need updated documentation to illustrate new functionality.

Security
--------

This change will require some validation of the permission model and
restrictions on the ceph storage provided by manila to ensure that accessing
a share doesn't break tenant restrictions.

Testing
-------

Unit tests will be added to each charm to cover any new functionality.

New functional tests will be added to each of the new and existing charms to
validate functionality, including end-to-end testing of the entire solution.

Dependencies
============

The packages that are required for this work are already included in the
Ubuntu archives in Universe. The required packages in Universe (ganesha-nfs)
will be proposed for inclusion to Main as a part of this work. There are no
other new, external dependencies. The first release that will be supported for
Manila with CephFS and Ganesha is Bionic Rocky.
