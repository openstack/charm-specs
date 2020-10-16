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

=======================
Cinder Ceph Replication
=======================

Cinder replication allows Disaster Recovery (DR) via the Ceph RBD driver.
During a failover scenario, the Cinder Ceph RBD driver will promote the
secondary Ceph images to primary and use those going forward.

The Ceph RBD driver implements the Cinder replication mechanism. More info
about it here:
https://docs.openstack.org/cinder/victoria/contributor/replication.html.

Problem Description
===================

There are two possible Ceph RBD mirroring modes (``pool`` and ``image``), and
the ``ceph-rbd-mirror`` charm knows to do only ``pool`` mode. More info about
RBD mirroring modes here:
https://docs.ceph.com/en/latest/rbd/rbd-mirroring/#enable-mirroring.

The ``image`` mirroring mode is required for the Ceph replication mechanism,
because the Ceph RBD driver will explicitly enable mirroring for each Ceph
image, and set the required image features (``exclusive-lock`` and
``journaling``). When mirroring mode is set to ``pool``, mirroring is enabled
by default on all the Ceph images in the pool, and the Ceph clients will not
take care of this part.

We want ``image`` mirroring mode (instead of ``pool``) for the Cinder
replication mechanism, because we don't want all the Cinder Ceph images
blindly mirrored. The mirroring will be orchestrated via the Cinder volume
types. More info about this here:
https://docs.openstack.org/cinder/ussuri/contributor/replication.html#volume-types-extra-specs.

Also, Cinder needs connection credentials to the remote cluster where the
images are mirrored, to be able to perform a failover. This information needs
to go into the ``cinder.conf``, under the backend section with replication
enabled.

Right now, if ``ceph-mon`` has a relation with ``ceph-rbd-mirror``, it
instructs its clients from the other relation (implementing the
``ceph-client`` interface) to use the ``exclusive-lock`` and ``journaling``
features, by default, for all the images in the pool.

We need to have the RBD mirroring features, enabled by default for all the
images, only when mirroring mode is set to ``pool``. When the mirroring mode
is set to ``image``, the Ceph client will explicitly set the RBD mirroring
features on each image, if needed.

Therefore, we need to ignore the advertised default RBD image features from
the ``ceph-mon`` charm, if the Ceph clients will have the mirroring mode set
to ``image``.

Proposed Change
===============

The ``cinder-ceph`` charm should have a new config option called
``rbd-mirroring-mode`` that should be passed as part of the ``create-pool``
broker request to the ``ceph-mon`` charm.

The ``ceph-mon`` charm will forward the broker request with the new
``rbd-mirroring-mode`` flag to the ``ceph-rbd-mirror`` charm, which should
be able to handle it and enable the requested mirroring mode for the pool.

Also, the ``ceph-rbd-mirror`` charm should use ``pool`` mirroring mode if it
doesn't find any mode explicitly set in the broker request. This
functionality should maintain the backwards compatibility of the charm.

On top of this, the Ceph clients implementing the new ``rbd-mirroring-mode``
config option (for example ``cinder-ceph`` charm) should not use the
advertised default RBD mirroring features from the ``ceph-mon`` charm, when
mirroring mode is ``image``. We can safely assume that these clients will
take care of explicitly setting those image features.

When all of the above are in place, Cinder needs to be able to connect to the
mirrored images, in the remote cluster, when a failover happens. It will
promote the mirrored image to primary and use that going forward.

The ``cinder-ceph`` charm will get the connection credentials to the remote
Ceph cluster through a new relation with the ``ceph-mon`` charm. The new
relation will implement the existing ``ceph-client`` interface, and it will
be named ``ceph-replication-device``.

Alternatives
------------

None

Implementation
==============

- cinder-ceph

  - Provides a new config option ``rbd-mirror-mode`` (with the ``pool`` as the
    default value), and it will send this to ``ceph-mon`` as part of the
    broker request to create the pool (which is going to be forwarded to
    ``ceph-rbd-mirror``).

  - Implements a new relation, using the ``ceph-client`` interface, with the
    ``ceph-mon`` charm. The relation is called ``ceph-replication-device``.
    This is going to be used to get the connection information to the remote
    Ceph cluster where the pool is mirrored. This information will go into the
    ``cinder.conf`` as ``replication_device`` config option, under the backend
    with replication enabled.

- ceph-mon

  - Forwards the broker requests to the ``ceph-rbd-mirror`` with information
    about the RBD mirroring mode.

- ceph-rbd-mirror

  - Handles the new RBD mirroring mode flag from the forwarded broker
    request given by ``ceph-mon`` charm.

- charm-helpers

  - Update the broker request handler to take into consideration the new
    ``rbd-mirroring-mode`` flag.

  - Update ``CephContext`` to ignore advertised default RBD features from the
    relation with ``ceph-mon``, if the client charm implements the new
    ``rbd-mirroring-mode`` flag, and that's set to ``image``.

  - Add Ceph relation name parameter to the ``CephContext`` constructor. The
    current implementation assumes that the clients will name their relation
    ``ceph``. This will remove the hard-coded value, and make it the default
    parameter value. Thus, we will maintain the backwards compatibility.

    This is going to be useful for charms that implement the ``ceph-client``
    interface, but the relation name is not ``ceph``.

Assignee(s)
-----------

1. ionutbalutoiu (primary)
2. oprinmarius

Gerrit Topic
------------

Use Gerrit topic "cinder-ceph-replication" for all patches related to this
spec.

.. code-block:: bash

    git-review -t cinder-ceph-replication

Repositories
------------

- openstack/charm-cinder-ceph

- openstack/charm-ceph-mon

- openstack/charm-ceph-rbd-mirror

- juju/charm-helpers

Documentation
-------------

The new ``rbd-mirroring-mode`` config option will be documented in the
``cinder-ceph`` charm, and in the Ceph RBD mirroring charm deployment guide.

Security
--------

- ``cinder-ceph``

  - It requires connection credentials to the remote Ceph cluster where the
    images are being mirrored by the Ceph RBD mirroring daemon.

    These are passed to ``cinder`` principal charm via the container-scoped
    relation. They will go into the ``cinder.conf``, under the backend with
    replication enabled.

Testing
-------

Code written or changed will be covered by unit tests; functional testing will
be implemented using the ``Zaza`` framework.

Dependencies
============

No new dependencies.
