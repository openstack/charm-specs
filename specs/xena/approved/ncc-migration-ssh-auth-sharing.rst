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

================================================================================
Sharing SSH authentication credentials across cloud-compute-related applications
================================================================================

Some cloud environments require different configuration settings
(e.g. allocation ratios) for different hosts, yet still need to allow
SSH-based migration between hosts. The proposed change in this spec
would allow sharing of credentials across multiple applications
related to the cloud-compute relation.

Problem Description
===================

At present, the nova-cloud-controller charm gathers SSH public keys
for all cloud-compute-related applications. These SSH keys are then
shared as needed, but only to peers within the same related
application.

While this is likely a sane default, especially considering that
different applications may represent different classes of compute
hosts which don't allow migration between them (e.g. SRIOV
vs. non-SRIOV), there are cases where there is no fundamental reason
to disallow migration.

In cases like this, at present it's necessary to manually modify
configuration files, perhaps setting them immutable, to provide the
different configuration values needed for the different types of
hosts, yet still allow them to perform migrations between each other.

Thus, it is believed it would be beneficial to allow for sharing all
SSH keys across all nova-compute applications to overcome those
limitations.

This specification is meant to address an implementation for the
following feature request on Launchpad:
https://bugs.launchpad.net/charms/+source/nova-cloud-controller/+bug/1468871

Proposed Change
===============

Share SSH keys across all nova-compute applications through the
cloud-compute relation with the nova-cloud-controller charm.

The code will now also have to go through each cloud-compute
relation whenever there is an update to a node or SSH key, in
order to keep the SSH keys on all compute nodes of all relations
consistent.

Upgrade Impact
--------------

Upon upgrading the nova-cloud-controller charm version, the
upgrade-charm hook code will cycle through all cloud-compute
relations and set the relation data with all SSH authorized
keys and known hosts to all compute nodes in the relations.

Performance Impact
------------------

Performance is slightly impacted as the hooks will take longer both
on the nova-cloud-controller charm to set a now higher number of
SSH keys and on each the nova-compute charms to process that higher
number of SSH keys received on the relation. The number of hooks
executed remains the same.

On cloud-compute-relation-changed hooks, when a change on SSH keys
is triggered by the compute node, or if a new compute node is added,
the code will also take slightly longer due to cycling through all
the relations to make all SSH keys on all compute nodes consistent,
where before it only needed to update the relation pertaining to the
compute node that triggered the change.

Alternatives
------------

One alternative may be to write a subordinate charm simply for the
purpose of sharing SSH keys and known hosts entries. While this could
in theory be done, it is less than ideal because that effectively
would create two sources of truth regarding which hosts should share
credentials with which hosts, and it would effectively replicate a
non-trivial amount of logic already handled by
charm-nova-cloud-controller.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Rodrigo Barbieri <rodrigo.barbieri@canonical.com>
  Paul Goins <paul.goins@canonical.com>

Gerrit Topic
------------

Use Gerrit topic "migration-ssh-auth-sharing" for all patches related to this spec.

.. code-block:: bash

    git-review -t migration-ssh-auth-sharing

Work Items
----------

* Allow for sharing across all cloud-compute-related applications

Repositories
------------

The changes would be isolated to the charm-nova-cloud-controller
repository.

Documentation
-------------

The release notes for the new charm release that includes this change
should mention the new behavior.

Security
--------

No security concerns are raised given that all the nova-compute charm
applications are part of the same environment and connected to the
same nova-cloud-controller nodes.

Testing
-------

This functionality can be tested via unit tests. Functional tests may
be overly cumbersome for this particular feature, although direct
testing in a lab environment should also be done as a sanity check.

Dependencies
============

N/A
