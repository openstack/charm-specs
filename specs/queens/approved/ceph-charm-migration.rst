..
  Copyright 2017 Canonical UK Ltd

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

====================================
Migrate Ceph Deployment Architecture
====================================

The all-in-one Ceph charm has been deprecated in favor of splitting out the
distinct functions of the Ceph monitor cluster and the Ceph OSDs. The Ceph
charm itself is receiving limited maintenance and all new features are being
added to the ceph-osd or ceph-mon charms. Unfortunately, there currently
exists no way of migrating existing deployments using the legacy architecture
to the preferred architecture.

Problem Description
===================

Deploying the all-in-one ceph charm was difficult to deploy properly and
often resulted in too many monitors deployed. This (and other) problems
eventually lead to splitting up the all-in-one ceph deployment into its
different compositional parts - ceph-mon and ceph-osd. New deployments are
encouraged to use the new architecture, but users which have deployed the
previous architecture have no way to migrate to the new architecture.

In order to migrate to the new architecture, users will need to:

#. Deploy the ceph-mon charm into the environment without bootstrapping a new
   monitor cluster.

#. Relate the ceph application to the new ceph-mon application.  The relation
   between these two applications is defined in the new `ceph-bootstrap`
   interface, documented herein.

#. Update the deployments using the ceph-client relation to point to the new
   ceph-mon application. This should include:

   - ceph-radosgw
   - nova-compute
   - cinder-ceph
   - cinder
   - cinder-backup
   - glance

#. Deploy the ceph-osd charm alongside the current ceph charms. The ceph-osd
   charm will not reformat any of the currently running OSD devices.

#. Remove the all-in-one ceph application.

Note: the configuration values will not be validated in any part of this
migration scenario. If the user deploys the ceph-osd application or the
ceph-mon application with different config values than that of the all-in-one
ceph charms, then there may be unexpected changes to the environment.

Proposed Change
===============

Most of the basic plumbing already exists to support the basic parts of this
migration. There are some parts that are actually missing an need attention
to detail.

Config Changes to charm-ceph-mon
--------------------------------

A new config option will be added to the ceph-mon charm indicating that the
charm should not do an initial bootstrap of the monitor cluster. When this
value is set to true, the charm will **not**:

#. bootstrap a new monitor cluster
#. automatically generate an fsid

This new configuration option will be defined as follows:

.. code-block:: yaml

    no-bootstrap:
      type: boolean
      default: False
      description: |
        Causes the charm to not do any of the initial bootstrapping of the
        Ceph monitor cluster. This is only intended to be used when migrating
        from the ceph all-in-one charm to a ceph-mon / ceph-osd deployment.
        Refer to the Charm Deployment guide at https://docs.openstack.org/charm-deployment-guide/latest/
        for more information.

New ceph-bootstrap Interface
----------------------------

Notably, there exists no way of adding a relation between the ceph-mon charm
and the all-in-one ceph charm. To address this, a new interface will be added
to the ceph and ceph-mon charms called `ceph-bootstrap`, enabling the
necessary information for joining the cluster to be shared. This is
essentially the same information that's shared on the peer `mon` interface for
the ceph charm itself.

Since the charms are not reactive, no new interface repository is required.
The information exchanged will contain the following:

.. code-block:: yaml

    ceph-bootstrap:
      - name: fsid
        type: string
        desc: |
          The fsid of the already bootstrapped monitor cluster

      - name: ceph-public-address
        type: string
        desc: |
          The public address that should be used by the charm for each of the
          units in the relation.

      - name: mon-key
        type: string
        desc: |
          The key used to authorize a monitor node for joining a ceph mon
          cluster.

In order to join this relation, the ceph-mon charm will require that the
``no-bootstrap`` config option be set to True and the monitor cluster is not
already bootstrapped locally. If either of these is not valid, the ceph-mon
charm will fail to join the relation.

Changes to charm-ceph
---------------------
In addition to implementing the `ceph-bootstrap` interface, the all-in-one
ceph charm needs to properly clean up when removing itself. It should not
remove any OSDs or Ceph packages as this would interrupt the operational ceph
cluster. However, the charm needs to remove its ceph.conf file as a
registered alternative during its stop hook.

Alternatives
------------

One alternative would be to **not** offer a migration to the new architecture
and leave the deployments stuck. This has rather unfortunate side effect
with respect to user happiness and has implications regarding the overall
lifetime of the ceph charm.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  james-page

Additional assignee(s):
  billy-olsen

Gerrit Topic
------------

Use Gerrit topic `charm-ceph-migration` for all patches related to this spec.

.. code-block:: bash

    git-review -t charm-ceph-migration

Work Items
----------

charm-ceph-mon
++++++++++++++
- Add new config flag
- Implement new bootstrap interface
- Remove additional monmap entries when removing the bootstrap relation

charm-ceph-osd
++++++++++++++
- Ensure that the charm supports multiple monitor relations

charm-ceph
++++++++++
- Add support for new bootstrap interface
- Remove registered alternative for ceph.conf in the stop hook
- Update charm README

charm-ceph-radosgw
++++++++++++++++++
- Ensure that the charm supports multiple monitor relations

charm-helpers
+++++++++++++
- Ensure that CephContext supports multiple monitor relations (consumed by
  charm-glance, charm-inder-ceph, charm-nova, etc)

openstack-charm-testing
+++++++++++++++++++++++
- Update all next.yaml and stable.yaml bundles to use split architecture
- Add legacy bundle which deploys an all-in-one ceph bundle

docs
++++
- Update charm deployment guide to clearly describe this process
- Update release notes with implementation note and reference to the
  deployment guide.

Repositories
------------

None

Documentation
-------------

This will need to be carefully documented in the following places:

#. Charm Deployment Guide
#. The Ceph Charm's README needs to refer to the migration process
   documentation and officially marked as deprecated.

Security
--------

No new security implications or this change.

Testing
-------

- Create an appropriate functional test that can be run at release gating
  (mojo, bundle tester, etc)
- Test the impact of running clients on the migration. This is important
  because the monitor information is exposed via the libvirt domain XML files
  which are created and may have an impact on running clients. Specifically,
  need to verify the impact to:
  - nova-compute w/ rbd backed instances
  - nova-compute w/ rbd cinder attached volumes
  - glance images w/ rbd backing

Dependencies
============

None
