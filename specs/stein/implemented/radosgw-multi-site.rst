..
  Copyright 2018 Canonical Ltd.

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
RadosGW Charm Multi-site Replication
====================================

Problem Description
===================

RadosGW `multi-site configuration <http://docs.ceph.com/docs/luminous/radosgw/multisite/>`__ can be set up to provide object sync for
disaster recovery and other purposes such as using the same object data stored
in a Ceph cluster local to a cloud region. A typical setup would look like
this:

* One zone per Zone Group (1 cluster per “region”);
* Multiple Zone Groups (“regions”);
* One Realm;
* Mode of operation: active-active or active-passive.

.. note::

    Ceph does support active-passive configurations, but to simplify
    deployment choice the charms will only support active-active.

There could also be more complex configurations with multiple zones (clusters)
per zone group.

In order to set this up, independent radosgw application deployments in
different Juju models have to be aware of each other and set up the
necessary configuration:

* Realm name for radosgw;
* Master zone group and master zone configuration;
* a system user for authentication between daemons;
* Access key and secret key setup for master zone authentication;
* A period needs to be updated after configuration changes to change an epoch.

.. note::

    Migration of an existing single site ceph-radosgw deployment to a
    multi-zone deployment will not be supported by the charms.

Proposed Change
===============

To be able to configure multi-site radosgw deployments it is necessary to
modify the radosgw charm to support cross-model relations between multiple
radosgw applications.  This relation will be used to exchange endpoint and
authentication information between the RADOS gateway deployment for
configuration of replication.

The charms will target a fix topology with a single realm and zone group
and two zones.  Its assumed that zones will be supported by separate
Ceph clusters but this is not a hard requirement (but is recommended).

Actions will be provided to promote and demote a RADOS gateway cluster
to and from master status. No automatic failover will be provided and
these operations must be performed by an operator in the event of site
failover/failback.

Alternatives
------------

As this is a RADOS gateway specific feature, no alternatives have been
considered.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

Gerrit Topic
------------

Use Gerrit topic "radosgw-multi-site" for all patches related to this spec.

.. code-block:: bash

    git-review -t radosgw-multi-site

Work Items
----------

* Implement support for new (cross-model) relation 'rgw-peer' between radosgw
  applications associated with different Ceph clusters.
* Add support for additional configuration keys to set up realm, zonegroup and
  zone for each ceph-radosgw deployment.
* Implement functionality to set up a master zone and add secondary zones to
  it.
* Write unit tests for newly added features.
* Write functional tests that include the deployment of multiple clusters and
  verification of object synchronization.

Repositories
------------

No new git repositories will be created.

Documentation
-------------

The ``radosgw`` charm README should contain instructions on deploying the
charm with new functionality enabled.

Security
--------

- TLS termination can be enabled on any side and needs to be supported without
  manual steps of synchronizing CA certificates between sites.  SSL CA certs
  will be shared between RADOS peers using the new rgw-peer relation.

Testing
-------

Code written or changed will be covered by unit tests; functional testing will
be done using ``Zaza``.

Dependencies
============

The ceph-radosgw charm currently uses the old-style radosgw systemd unit and
a global cephx key for access to the underlying Ceph cluster.

The charm should be migrated to use the new ceph-radosgw systemd units and
switch to use of cephx keys which are specific to individual radosgw units.
