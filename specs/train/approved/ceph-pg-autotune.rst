..
  Copyright <YEARS> <HOLDER>  <--UPDATE THESE

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===============================
Ceph PG Auto Tuning
===============================

In the Nautilus release, Ceph has introduced placement group (PG) autotuning.
The charms should support this module allowing a Ceph cluster to grow or shrink
naturally.

Problem Description
===================

If an existing cluster is incorrectly sized at deploy time, it is very
difficult for an operator to resolve the imbalance. PG Autotuning was
designed to remove the black-magic of PG tuning.

Proposed Change
===============

A new configuration option (pg-autotune) will be introduced to enable the new
pg_autoscaler module. This config option will have no effect for Ceph releases
before Nautilus and will have no default.

In a new Ceph deployment with a release greater than Nautilus, a null value for
'pg-autotune' will cause the autoscaler module to be enabled and configured on
all pools, with the global default additionally configured as ON. In an upgrade
to Nautilus, a null value will not cause the autoscaler module to be loaded,
nor will pool metadata change.

For an existing deployment to enable the autoscaler, an administrator should
change the config value to 'True', after which the charm will process the new
option, as well as updating the existing pools as necessary.

The ceph-mon charms already handle sizing the default placement groups via the
expected ratios so the required data is already present.

Alternatives
------------

The current solution is a viable alternative, as long as the expectations about
pool sizing is correct at deploy time. Other alternatives include manual
operator intervention to resolve an incorrectly sized pool.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  chris.macnaughton

Gerrit Topic
------------

Use Gerrit topic "pg-autotune" for all patches related to this spec.

.. code-block:: bash

    git-review -t pg-autotune

Work Items
----------

- Enable the pg_autoscaler manager module if the Ceph version is
  Nautilus or higher
- Disable the autoscaler globally, by default
- Enable the autoscaler and configure pools when toggled via configuration

Repositories
------------

No new repositories will need to be created.

The charms.ceph and charm-ceph-mon repositories will be the primary focus of
this development.

Documentation
-------------

There will be a new configuration option added to toggle on the
autoscaler for the Ceph cluster. This configuration option should additionally
be called out in the README for the ceph-mon charm.

Security
--------

There are no expected security risks associated with this change.

Testing
-------

An additional test-case will be added enabling this config option to ensure
that the functionality works as expected. Additionally, the functionality will
be validated via inspection of Ceph's suggestions for auto-tuning.

Dependencies
============

There are no new dependencies.