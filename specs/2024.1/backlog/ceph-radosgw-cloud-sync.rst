..
  Copyright 2023, Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===================================
Ceph RADOS Gateway (RGW) Cloud Sync
===================================

Ceph RGW has a module called ``Cloud Sync`` which allows syncing zone data to
a remote cloud service. The sync is unidirectional, data is not synced back
from the remote zone. The goal of this module is to enable syncing data to
multiple cloud providers. The currently supported cloud providers are those
that are compatible with AWS (S3).

More info about Ceph ``Cloud Sync`` here:
https://docs.ceph.com/en/latest/radosgw/cloud-sync-module.

The ``Cloud Sync`` module is built atop of the multi-site framework that allows
for forwarding data and metadata to a different external tier.

More info about Ceph sync modules here:
https://docs.ceph.com/en/quincy/radosgw/sync-modules.

Problem Description
===================

The current ``ceph-radosgw`` charm does not support the ``cloud-sync`` module.
Given the fact that the ``cloud-sync`` module is built atop of the multi-site
framework, we can leverage the existing ``radosgw-multisite`` Juju relation
interface.

The ``cloud-sync`` module is enabled via a new relation with the primary
ceph-radosgw application. The deployment is similar with the existing RGW
multi-site replication steps:
https://ubuntu.com/ceph/docs/setting-up-multi-site.

The only differences being:

- Both ``primary-ceph-radosgw`` and ``secondary-ceph-radosgw`` (with the
  ``cloud-sync`` enabled zone) are related to the same Ceph cluster.

- When data is replicated from ``primary-ceph-radosgw`` zone, the
  ``secondary-ceph-radosgw`` zone will write into a remote S3, instead of Ceph
  storage. The ``secondary-ceph-radosgw`` zone will have the appropriate S3
  credentials configured for this task.

- Data sync is unidirectional, therefore the ``secondary-ceph-radosgw`` zone
  will be read-only.

More info about how to configure the ``cloud-sync`` module here:
https://docs.ceph.com/en/latest/radosgw/cloud-sync-module/#how-to-configure.

Proposed Change
===============

Add a new relation, called ``cloud-sync``, to the ``ceph-radosgw`` charm. The
new relation will implement the existing ``radosgw-multisite`` relation
interface.

In the new ``cloud-sync`` relation, when the secondary multi-site secondary
zone is created, we need to pass ``--tier-type=cloud`` to the
``radosgw-admin zone create`` command in order to have the ``cloud-sync``
module enabled. Besides this, we need to add the S3 target credentials via
``--tier-config`` parameter of the ``radosgw-admin zone modify`` command.

These steps are documented at:
https://docs.ceph.com/en/latest/radosgw/cloud-sync-module.

The Ceph ``cloud-sync`` module allows multiple S3 targets to be configured in
the same zone tier config. For this, we have ``profiles`` in the tier config.
Each profile maps a single source bucket (or multiple buckets via prefix) to
one S3 destination (Many-To-One mapping). The ``profiles`` in the tier config
are optional.

A profile contains info about:

- ``source_bucket``, either a bucket name, or a bucket prefix (if ends with
  ``*``) that defines the source bucket(s) for this profile.

- ``target_path``, a string that defines how the target path is created. The
  target path specifies a prefix to which the source object name is appended. The
  target path is configurable, and it can include any of the following variables:

  - ``$sid``: unique string that represents the sync instance ID
  - ``$zonegroup``: the zonegroup name
  - ``$zonegroup_id``: the zonegroup ID
  - ``$zone``: the zone name
  - ``$zone_id``: the zone id
  - ``$bucket``: source bucket name
  - ``$owner``: source bucket owner ID

  For example: ``target_path = rgwx-${zone}-${sid}/${owner}/${bucket}``

- ``connection_id``, ID of the connection (with credentials) that will be used
  for this profile.

A new charm config, called ``cloud-sync-target-path``, will be added
to configure the target path for all the profiles. This allows a consistent
target path for the configured ``cloud-sync`` zone.

The profiles are configured through the use of ``s3-integrator`` Juju applications
together with the new config option ``cloud-sync-target-path``.

It is mandatory to have a default S3 target for all the buckets that don't
have a profile configured. The rationale is that every bucket needs to have
a sync target, and the default target is the fallback for any bucket that
doesn't have a profile configured. A new charm config will be added, called
``cloud-sync-default-s3-target`` for this purpose.

It is obvious that we need to handle S3 credentials for the S3 targets
configured in the ``cloud-sync`` zone. For this purpose, we will use the
``s3-integrator`` charm (https://github.com/canonical/s3-integrator). The
``ceph-radosgw`` charm will have new relation with the ``s3-integrator``
charm.

Each deployed application of the ``s3-integrator`` charm will handle
credentials for a single S3 target. When relating multiple ``s3-integrator``
applications to the same ``secondary-ceph-radosgw`` cloud-sync application,
the tier config will be updated with profiles for each S3 target.

For example, the following Juju deployment commands:

.. code-block:: bash

    #
    # Assuming ceph-mon is already deployed
    #
    juju deploy ceph-radosgw primary-ceph-radosgw \
      --config realm=eu \
      --config zonegroup=east \
      --config zone=primary

    juju relate ceph-radosgw:mon ceph-mon:radosgw

    juju deploy ceph-radosgw secondary-ceph-radosgw \
      --config realm=eu \
      --config zonegroup=east \
      --config zone=primary-cloud-sync \
      --config 'cloud-sync-target-path=${bucket}' \
      --config cloud-sync-default-s3-target=minio-dev

    juju relate secondary-ceph-radosgw:mon ceph-mon:radosgw

    juju deploy s3-integrator minio-dev \
      --config endpoint=http://10.7.133.248:9000 \
      --config region=us-east-1 \
      --config s3-uri-style=path

    juju deploy s3-integrator minio-production \
      --config endpoint=http://10.7.133.250:9000 \
      --config region=us-east-2 \
      --config s3-uri-style=path \
      --config 'bucket=production*'

    juju relate ceph-radosgw-cloud-sync:s3-credentials minio-dev:s3-credentials
    juju relate ceph-radosgw-cloud-sync:s3-credentials minio-production:s3-credentials

    #
    # After all applications' units are idle
    #
    juju relate ceph-radosgw-cloud-sync:cloud-sync ceph-radosgw:primary

    juju run minio-dev/leader sync-s3-credentials --string-args access-key=MY_DEV_ACCESS_KEY secret-key=MY_DEV_SECRET_KEY
    juju run minio-production/leader sync-s3-credentials --string-args access-key=MY_PROD_ACCESS_KEY secret-key=MY_PROD_SECRET_KEY

will render the following tier config in the cloud sync zone:

.. code-block:: JSON

    {
        // ...
        "name": "primary-cloud-sync",
        // ...
        "tier_config": {
            "connections": [
                {
                    "id": "minio-dev",
                    "endpoint": "http://10.7.133.248:9000",
                    "region": "us-east-1",
                    "host_style": "path",
                    "access_key": "MY_DEV_ACCESS_KEY",
                    "secret": "MY_DEV_SECRET_KEY"
                },
                {
                    "id": "minio-production",
                    "endpoint": "http://10.7.133.250:9000",
                    "region": "us-east-2",
                    "host_style": "path",
                    "access_key": "MY_PROD_ACCESS_KEY",
                    "secret": "MY_PROD_SECRET_KEY"
                }
            ],
            "profiles": [
                {
                    "connection_id": "minio-production",
                    "source_bucket": "production*",
                    "target_path": "${bucket}"
                }
            ],
            "connection_id": "minio-dev",
            "target_path": "${bucket}"
        },
        // ...
    }

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee: ionutbalutoiu

Gerrit Topic
------------

Use Gerrit topic "ceph-radosgw-cloud-sync" for all patches related to this
spec.

.. code-block:: bash

    git-review -t ceph-radosgw-cloud-sync

Work Items
----------

- Add two new charm configs to ``ceph-radosgw``:

  - ``cloud-sync-default-s3-target``, the default S3 target for buckets that
    don't have a profile configured in the tier config.

  - ``cloud-sync-target-path``, string that defines how the target path is
    created. The target path specifies a prefix to which the source object
    name is appended.

- Add a new relation called ``cloud-sync`` to the ``ceph-radosgw`` charm.
  The new relation implements the existing ``radosgw-multisite`` interface.
  The cloud-sync secondary zone will be configured with ``--tier-type=cloud``,
  and connection info for the S3 targets will be fetched from the relation
  with the ``s3-integrator`` charm.

  When the ``cloud-sync`` relation is established, the ``ceph-radosgw``
  cloud-sync application will be blocked until a relation with the
  ``s3-integrator`` application is created, which provides S3 credentials for
  the configured ``cloud-sync-default-s3-target``.

- Add a new relation called ``s3-credentials``, implementing ``s3`` interface,
  used to fetch S3 credentials for each S3 target in the ``cloud-sync`` tier
  config.

  The name of related ``s3-integrator`` application will be used as the
  profile name configured in the tier config. From the relation data, we also
  fetch the source bucket(s) for each profile.

Repositories
------------

- https://opendev.org/openstack/charm-ceph-radosgw

Documentation
-------------

The config options (``cloud-sync-default-s3-target`` and
``cloud-sync-target-path``) will be documented in the ``ceph-radosgw`` charm.

Also, additional documentation to charm deployment guide should be added for
the new ``cloud-sync`` relation.

Security
--------

- ``ceph-radosgw``

  - The Ceph ``Cloud Sync`` module requires S3 connection credentials for the
    configured S3 targets. These credentials are fetched from the
    ``s3-credentials`` relation with an application that implements the ``s3``
    relation interface.

Testing
-------

Code written or changed will be covered by unit tests; functional testing will
be implemented using the ``Zaza`` framework.

Dependencies
============

No new dependencies.
