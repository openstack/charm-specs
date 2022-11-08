..
  Copyright 2018 Canonical Ltd.

=======================================
Swift Backup Driver for Cinder
=======================================

It is possible to use swift API for cinder backup purposes. In one of the
configurations the swift API provider is deployed via non-Juju means. This
spec is aimed to address this use-case specifically.

Problem Description
===================

Cinder has support for backing up volumes via Swift API. In order to use it
a URL is needed along with authentication details.

The following methods are supported to use to authenticate against a swift
endpoint:

* v2 auth (URL, username, password, tenant);
* v3 auth (URL, username, password, project domain, project, user domain,
  username, password);

The backup to swift functionality is supported on all versions of OpenStack
supported at the time of writing. It was `added in 2013.1 <https://blueprints.launchpad.net/cinder/+spec/volume-backups>`__

Backup configuration should be added to cinder.conf and can be propagated from
the cinder-backup-like charm to the cinder charm. Extending the existing
cinder-backup charm is not favorable as it is ceph-specific.

Config options for the swift backend are `present here <https://docs.openstack.org/cinder/rocky/configuration/block-storage/backup/swift-backup-driver.html>`__

Proposed Change
===============

Implement a new subordinate charm in reactive framework. This charm will be
deployed alongside cinder deployments. This will be a subordinate charm to
cinder, that installs cinder-backup and configures cinder to use swift as
a backend for volume backups.

* config options for configuring the swift backend.
  Following configs need to be modeled in the charm.

  - backup_swift_url

  - backup_swift_auth_url

  - backup_swift_user

  - backup_swift_key

  - backup_swift_auth_version

  - backup_swift_tenant

  - backup_swift_user_domain

  - backup_swift_project_domain

  - backup_swift_project

  - backup_swift_container

  - backup_swift_object_size

  - backup_swift_block_size

  - backup_swift_ca_cert_file

Implement a new interface cinder-backup in reactive framework for sending
cinder-backup external backend configuration.

Alternatives
------------

* extend cinder-backup to support swift options (discarded as it is already
  Ceph-specific);
* extend cinder-backup to support clusters not deployed via charms by creating
  proxy charms (discarded as cinder-backup is Ceph-specific and proxy charms
  add additional complexity).

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  yoshikadokawa
  alitvinov

Gerrit Topic
------------

Use Gerrit topic "cinder-backend-swift" for all patches related to this spec.

.. code-block:: bash

    git-review -t cinder-backend-swift

Work Items
----------

* Implement a new reactive subordinate charm, cinder-backup-swift
* Implement a new reactive charm interface, cinder-backup
* write unit tests;
* write functional tests using the zaza test framework (a functional tests
  should include deploying swift via swift-proxy and swift-storage charms with
  config options used to connect to it from the cinder-backend-swift charm).

Repositories
------------

The existing classic-style charm in the following repository:

.. code-block:: bash

    openstack/charm-cinder-backup-swift

will be replaced by the new reactive charm.

New git repository:

.. code-block:: bash

    openstack/charm-interface-cinder-backup


Documentation
-------------

The charm should contain documented options and authentication methods for the
target Swift cluster.

Security
--------

The Swift endpoint might be TLS-terminated, therefore, an option to
provide a CA certificate is required.

Testing
-------

* unit tests;
* functional tests (as mentioned above).

Dependencies
============

- This charm will support OpenStack Queens and
  Ubuntu 18.04 Bionic as its baseline
- A new project will need to be created based on the OpenStack Project
  `Creator's Guide <https://docs.openstack.org/infra/manual/creators.html>`__
