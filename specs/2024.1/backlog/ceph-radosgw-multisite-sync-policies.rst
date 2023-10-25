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

=================================================
Ceph RADOS Gateway (RGW) Multi-site Sync Policies
=================================================

Multi-site sync policies provide fine-grained control of data movement
between buckets in different zones. The sync policy can be configured at the
zonegroup level, but it can also be configured at the bucket level.

By default, no sync policies are configured in the multi-site scenario, and
all the buckets are synced from the primary zone to the secondary zones.

The goal is to allow selective sync of buckets between zones. This is
important, for security purposes, when configuring secondary zones in
the multi-site scenario.

More info about multi-site sync policies here:
https://docs.ceph.com/en/latest/radosgw/multisite-sync-policy.

Problem Description
===================

The current ``ceph-radosgw`` charm does not support configuration of multi-site
sync policies. At the moment, we do not have the option to stop particular
buckets syncing from primary zone to secondary zones.

Adding sync policies to the ``ceph-radosgw`` charm will give us more control
over how the data is synced from primary RGW zone to secondary RGW zones. This
can be generally useful for syncing only essential buckets to the secondary
cluster. It is particularly valuable for one-way data synchronizations.

Proposed Change
===============

Allow configuration of a default zonegroup sync policy.

This would allow all the buckets sync disabled by default, in the zonegroup.
And, for particular buckets, having the possibility to enable syncing to
secondary zones via bucket level sync policies. This is very useful when
having selective data sync to secondary RGW zones.

By default, the ``ceph-radosgw`` charm will not configure any sync policies
to maintain backwards compatibility with the existing functionality.

Sync policies can be created by the command ``radosgw-admin sync group
create``. A sync policy group can be in three states:

- ``enabled``, sync is allowed and enabled.

- ``allowed``, sync is allowed (but not enabled).

- ``forbidden``, sync is not allowed.

In the sync policy group, we need to configure:

- data-flows, via command ``radosgw-admin sync group flow create``.

- pipes, via command ``radosgw-admin sync group pipe create``.

A data-flow defines the flow of data between the different zones, and it can
define:

- ``symmetrical`` data flow, in which multiple zones sync data from each other.

- ``directional`` data flow, in which the data moves in one way from one zone
  to another.

A pipe defines the actual buckets that can use these data flows, and the
properties that are associated with it (for example: source object prefix).

Alternatives
------------

None

Implementation
==============

Two new configs will be added to the ``ceph-radosgw`` charm:

- ``sync-policy-state``, allows to configure the state of the default sync
  policy group (``enabled``, ``allowed``, or ``forbidden``). This will be
  configured in the primary ``ceph-radosgw`` application, and it will dictate
  the default zonegroup sync policy. This config will be empty, by default.
  If it's empty, sync policies are not configured in the zonegroup. This way,
  we maintain backwards compatibility with the existing functionality of the
  charm.

- ``sync-policy-flow-type``, defines the data flow type that will be
  configured in the default sync policy group between primary zone and a
  secondary zone. This config will be ``symmetrical`` by default, and it is
  used only when the primary ``ceph-radosgw`` application has config option
  ``sync-policy-state`` non-empty, and multi-site sync policies are configured.

Implement the ``primary-relation-changed`` hook, where we configure the
default sync policy (if the ``sync-policy-state`` config is set). If the
default policy is created:

- A data flow with each secondary zone is configured. The data flow type is
  obtained from the ``sync-policy-flow-type`` config passed on the relation
  data by the secondary ``ceph-radosgw`` applications.

- A pipe with each secondary zone is configured. The pipe allows all the
  buckets to use the previously defined data flow.

Three Juju actions will be added to the ``ceph-radosgw`` charm:

- ``enable-buckets-sync``, takes a comma-separated list of buckets to enable
  sync with bucket level sync policy. This is meant to be used in conjunction
  with ``allowed`` ``sync-policy-state`` config, which allows buckets syncing,
  but it's disabled by default.

- ``disable-buckets-sync``, takes a comma-separated list of buckets to forbid
  sync with bucket level sync policy. This is useful when you want to disable
  syncing for some buckets only, regardless of the default zonegroup sync
  policy.

- ``reset-buckets-sync``, takes a comma-separated list of buckets to reset
  bucket level sync policy. After this is executed, the buckets will be synced
  according to the default zonegroup sync policy.

If the Juju actions are executed on the primary ``ceph-radosgw`` application,
any bucket level sync policies are automatically propagated from the primary
RGW site to all the secondary RGW sites.

On the other hand, if we execute the Juju actions on the secondary
``ceph-radosgw`` applications, the bucket level sync policies are not
propagated to any other RGW site.

All these new Juju actions can be executed only on the primary ``ceph-radosgw``
application. They will fail if they are executed on the secondary
``ceph-radosgw`` applications.

Assignee(s)
-----------

Primary assignee: ionutbalutoiu

Gerrit Topic
------------

Use Gerrit topic "ceph-radosgw-multisite-sync-policies" for all patches
related to this spec.

.. code-block:: bash

    git-review -t ceph-radosgw-multisite-sync-policies

Work Items
----------

- Add two new charm configs to ``ceph-radosgw``:

  - ``sync-policy-state``, which allows to configure the state of the default
    sync policy group.

  - ``sync-policy-flow-type``, defines the type of data flow that will be
    configured in the default sync policy group between the primary zone and a
    secondary zone.

- Implement the ``primary-relation-changed`` hook with all the details
  described in the ``Implementation`` section.

- Add three Juju actions to the ``ceph-radosgw`` charm:

  - ``enable-buckets-sync``, takes a comma-separated list of buckets to enable
    sync with bucket level sync policy.

  - ``disable-buckets-sync``, takes a comma-separated list of buckets to forbid
    sync with bucket level sync policy.

  - ``reset-buckets-sync``, takes a comma-separated list of buckets to reset
    bucket level sync policy.

Repositories
------------

- https://opendev.org/openstack/charm-ceph-radosgw

Documentation
-------------

The new charm configs, and Juju actions will be documented in the
``ceph-radosgw`` charm.

Also, additional documentation to charm deployment guide should be added for
the new optional multi-site sync policies.

Security
--------

If used, this will improve the security of multi-site Ceph RGW deployment.

Testing
-------

Code written or changed will be covered by unit tests; functional testing will
be implemented using the ``Zaza`` framework.

Dependencies
============

No new dependencies.
