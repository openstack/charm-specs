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

=====================
Ceph RBD Mirror Charm
=====================

Problem Description
===================

RBD image mirroring can be used to provide a solution for Ceph cluster disaster
recovery. Ceph has a daemon called
`rbd-mirror <http://docs.ceph.com/docs/mimic/rbd/rbd-mirroring/>`__ which can
be placed on a primary and a backup cluster and provide asynchronous
replication of RBD images in a given pool.

rbd-mirror can work in two modes:

* pool (all images in a given pool are synchronized);
* image (per-image synchronization).

The scenario targeted by this spec involves an operator when it comes to
performing promote/demote actions and the DR procedure is operator-driven
as opposed to a full automatic failover in the event of a outage at the
primary site.

.. note::

    Promotion/demotion of pools will be operator driven

.. note::

    RADOS objects are not mirrored so for mirroring radosgw objects which
    use RADOS objects or gnocchi metrics stored in RADOS objects different
    backup mechanisms are required - this spec covers RBD images only.

.. note::

    RBD mirroring relies on the use of the exclusive-lock and journaling
    features of RBD; these are only supported in the userspace integration
    libraries as used by libvirt and qemu for native KVM virtualization.
    This requirement excludes the use of this feature with LXD based clouds
    which disable the majority of RBD features for compatibility with the
    Linux kernel RBD driver.

.. note::

    The initial RBD mirror charm will only support mirroring of whole
    pools.

Proposed Change
===============

High Level Design
-----------------

As rbd-mirror is a separate package and the service itself acts as an RBD
client it makes sense to implement the target functionality in a separate
principle charm (ceph-rbd-mirror). The charm will accept parameters on which
pools to replicate and be able to be related to multiple ceph-mon applications
in separate clusters.

The charm will relate to a local ceph cluster and a remote ceph cluster
typically using a cross model relation.

A new interface type ('rbd-mirror') will be created to support this
integration; this will be provided by the ceph-mon charm, and consumed by
the new ceph-rbd-mirror charm for both local and remote cluster connections.

Each rbd-mirror daemon requires a key for connectivity to the local cluster
(named uniquely for the daemon) and a key for connectivity to the remote
cluster (named globally for all rbd-mirror daemons).  Multiple ceph
configurations will also be maintained on the ceph-rbd-mirror units -
'ceph' to reference the local cluster and 'remote' to reference the
remote cluster.  Configuration files and keys will be prefixed inline
with this naming - for example:

.. code::

    $ ls /etc/ceph
        ceph.conf
        ceph.client.rbd-mirror.<hostname>.keyring
        remote.conf
        remote.client.rbd-mirror.keyring

In order to support resilience and scale-out of rbd mirroring, multiple
units of the charm may be deployed; as a result this feature will only
be supported with Ceph Luminous or later (which support multiple instances
of the rbd-mirror service).

Deployment and Scalability Considerations
-----------------------------------------

From the deployment perspective the charm units should have high-bandwidth
and low-latency L3 connectivity to access and replication networks of both
clusters to be able to keep up with the changes to Ceph pools it tries to
replicate. At minimum, static routes will need to be configured on the node
running rbd-mirror daemon but that is outside of the scope of this spec.

Multiple units of the ceph-rbd-mirror charm may be used to scale out
replication traffic.

Alternatives
------------

No alternative solutions have been considered.

Dependencies
============

This feature relies on use of a Juju version which supports cross model
relations.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

  <tbd>

Gerrit Topic
------------

Use Gerrit topic "rbd-mirror" for all patches related to this spec.

.. code-block:: bash

    git-review -t rbd-mirror

Work Items
----------

* Implement a new reactive charm called ceph-rbd-mirror.
* Implement the following relation:
  * rbd-mirror - ceph-mon (local and cross model).
* Add "cluster" endpoint to extra-bindings in metadata.yaml to allow binding
  the "cluster" endpoint to a ceph replication space.
* ceph-mon relations should retrieve cluster details and cephx keys via the
  broker protocol implemented in Ceph charms (code reuse).
* Add config options to specify pool names for replication.
* Automate creation of pools on a backup cluster if they are not present.
* Add actions to promote and demote pools.
* Enable RBD journaling feature as documented in rbd-mirror docs.
* Write unit tests.
* Write functional tests via zaza framework.

Repositories
------------

A new git repository will be required for the ``ceph-rbd-mirror`` charm:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-ceph-rbd-mirror

Documentation
-------------

The ``ceph-rbd-mirror`` charm should contain a README with instructions on
deploying the charm and on limitations related to scalability and networking.

Security
--------

- Users created for replication must not have admin privileges - they only
  need to be able to write to the pools they require on the target cluster.
  This is supported through the existing group based permissions system
  in the ceph-mon broker using the 'rbd' profile for mon and osd permissions.

Testing
-------

Code written or changed will be covered by unit tests; functional testing will
be done using ``Zaza``.
