..
  Copyright 2021 Canonical Ltd.

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
Deferred Service Restarts
=========================

When a charm or package is upgraded services are restarted. This can impact the
data plane causing outages. This spec is aimed at giving the cloud operator the
option of deferring these restarts to a later, more convenient, time.

Problem Description
===================

One example of the issue would be a charm upgrade which includes a minor
template change. Charm upgrades are executed on all units of an application
simultaneously (the cloud operator has no control over this). The charms are
designed to restart services when configuration files are changed so the same
set of services, across all units, will be simultaneously restarted.

Another example is when a package update is triggered by the charm, it may be
useful to defer any service restarts until after a lengthy package update
completes.

Proposed Change
===============

Add charm configuration option to control automatic service restarts.

.. code:: yaml

    enable-auto-restarts:
      type: boolean
      default: True
      description: |
        Allow the charm and packages to restart services automatically when
        required

Add charm action to run service restarts.

.. code:: yaml

    restart-services:
      description: |
        Restarts services this charm manages.
      params:
        deferred-only:
          type: boolean
          default: false
          description: |
            Only restart deferred services.
        services:
          type: string
          default: ""
          description: |
            Optional space seperated list of services to restart. If unset
            then it applies to all services the charm controls.
    show-deferred-restarts:
        description: |
            Show the outstanding restarts


If a charm has `enable-auto-restarts` disabled this will be reflected in the
charms workload status message. The starts that have been requested, but
deferred, by either the charm or a package update can be seen by running
the `show-deferred-restarts` action.

To block restarts, charms will first check whether `enable-auto-restarts`
has been disabled and only perform the restart if permitted. Ideally preventing
services from being stopped or restarted would be done within systemd but this
does not seem to be possible in a clean way. Relying on the charm to police
itself is not ideal as there are many places a restart command can emanate from
and there is always the risk that a future charm update will introduce a
restart call that does not honour `enable-auto-restarts`.

Packages may try and restart a service during an update. To stop this from
happening a custom `/usr/sbin/policy-rc.d` script will be installed to block
the package changes from causing any restarts. Charms will indicate which
services they want to block restarts for by writing a file to
`/etc/policy-rc.d`.  `/usr/sbin/policy-rc.d` will collect all the files from
`/etc/policy-rc.d` to construct a list of services which are not permitted to
be restarted.

The flow for deferring restarts:

#. Operator updates application to set `enable-auto-restarts=False`
#. A `/usr/sbin/policy-rc.d` file is written
#. A charm specific file is placed in `/etc/policy-rc.d` which list which
   services should be blocked from restarting.

The flow for re-enabling restarts:

#. Operator updates application to set `enable-auto-restarts=True`
#. The charm specific file is removed from `/etc/policy-rc.d`
#. Optionally, the operator runs the `restart-services` action against each
   application unit.

The existing `Pause` and `Resume` actions will continue to operate as they
do now irrespective of the value of `enable-auto-restarts`

Alternatives
------------

The problem could be handled if Juju were to implement a feature allowing charm
updates to be performed on a unit by unit basis.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gnuoy

Gerrit Topic
------------

Use Gerrit topic "deferred-restarts" for all patches related to this spec.

.. code-block:: bash

    git-review -t deferred-restarts

Work Items
----------

* Write `policy-rc.d` script and add to charm-helpers
* Write functions in charm-helpers for managing the masking of services
  and files in `/etc/policy-rc.d`.
* Each charm will need to add the new action and configuration option and the
  charm will need to gate any service interupting restarts on the value of
  `enable-auto-restarts` in a similar way to the existing `is_unit_paused_set`

Repositories
------------

No new repositories are required.

Documentation
-------------

The new actions will need to be documented in the charm-guide and the upgrade
sections updated.

Security
--------

None

Testing
-------

The regularly scheduled upgrade testing would be a good place to test this
feature.

Dependencies
============

None
