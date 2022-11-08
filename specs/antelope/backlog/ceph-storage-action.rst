..
  Copyright 2016, Canonical UK

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
Ceph Storage Action
===============================

We should allow the user to specify the class of storage that a given
storage device should be added to. This action would allow adding a list
of osd-devices into specified Ceph buckets.

Problem Description
===================

All osd-devices are currently added to the same default bucket making it
impossible to use multiple types of storage effectively. For example, users
may wish to bind SSD/NVMe devices into a fast/cache bucket, 15k spindles into
a default bucket, and 5k low power spindles into a slow bucket for later
use in pool configuration.

Proposed Change
===============

Add an action that includes additional metadata into the osd-devices list
allowing for specification of bucket types for the listed devices. These OSDs
would be added to the specified bucket when the OSD is created.

Alternatives
------------

- User could specify a pre-build usage profile and we could map device types
  onto that profile by detecting the device type and deciding, based on the
  configured profile, what bucket the device should go into.

  This solution is being discarded because of the difficulty of designing
  and implementing usage profiles to match any use case automigically. It will
  also require a lot of work to correctly identify the device type and decide
  where to place it in the desired profile.

  In addition, it would require that we only support a single "profile" within
  a deployed ceph-osd cluster.

- Charm could define additional storage attach points in addition to
  osd-devices that would allow the user to specify what bucket they
  should add devices to.

  The reason for discarding this solution is that it was determined to be too
  limiting because of only supporting a fixed number of bindings, in addition
  to making changes harder because of backwards compatibility requirements.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Chris MacNaughton <chris.macnaughton@canonical.com>

Contact:
  Chris Holcombe <chris.holcombe@canonical.com>

Gerrit Topic
------------

Use Gerrit topic "storage-action" for all patches related to this spec.

.. code-block:: bash

    git-review -t storage-action

Work Items
----------

1. Create a bucket with the required name. We do this so that we can create
   pool in the specified bucket type. This would be handled within the ceph-mon
   charm.
2. Create an action that returns a list of unused storage devices along with
   their device types.
3. Create an action that takes `osd-devices` and `storage-type`, that would
   be an enum into the buckets that we create. Rather than be a user specified
   string, this would be an enumerated list in the ceph charms provided through
   the shared charms_ceph library.
4. Add the ability for other charms to request the bucket that should back
   their created pools when creating pools through the broker.

Documentation
-------------

This will require additional documentation be added around how to use the
action correctly and what to expect from it. This documentation will be added
to the charm README

Security
--------

This should have no security impact.

Testing
-------

There will need to be new unit tests and functional tests to ensure that
the necessary buckets are created and that the disks are added to them.

Repositories
------------

None

Dependencies
============

None
