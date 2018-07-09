..
  Copyright 2017 Canonical UK Ltd

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=================================
Extended Swift Cluster Operations
=================================

The Swift charms currently support a subset of operations required to support
a Swift cluster over time. This spec proposes expanding on what we already have
in order to support more crucial operations such as reconfiguring the
rings post-deployment.

Problem Description
===================

To deploy a Swift object store using the OpenStack charms you are required to
deploy both the swift-proxy and swift-storage charms. The swift-proxy charm
performs two key roles - running the api endpoint service and managing the
rings - while the swift-storage charm is responsible for the running the swift
object store services (account, container, object).

As they stand, these charms currently support a base set of what is required to
effectively maintain a Swift cluster over time:

* deploy swift cluster with configurable min-part-hours, replica count,
  partition-power and block storage devices.

* once deployed the only changes that can be made are the addition of
  block devices and modification of min-part-hours. Changes to
  partition-power or replicas are ignored by the charm once the rings
  have already been initialised.

This forces operators to manually apply changes like adjusting the
partition-power to accomodate for additional storage added to the cluster. This
poses great risk since manually editing the rings/builders and syncing them
across the cluster could easily conflict with the swift-proxy charm's native
support for doing this resulting in a broken cluster.

Proposed Change
===============

The proposal here is to extend the charm support for ring management in order
to be able to support making changes to partition-power, replicas and possibly
others and have the charm safely and automatically apply these changes and
distribute aross the cluster.

Currently we check whether the rings are already initialised and if they are
we ignore the partition power and replica count configured in the charm i.e.
changes are not applied. To make it possible to do this we will need to remove
these blocks and implement the steps documented in [0] and [1]. I also propose
that charm impose a cluster size limit (number of devices) above which we
refuse to make changes until the operator has paused the swift-proxy units i.e.
placed them into "maintenance mode" which will shutdown the api services and
block any restarts until the units are resumed. The user will also have the
option to set disable-ring-balance=true if they want check that their changes
have been applied successully (to the builder files) prior to having the rings
rebuilt and sycned across the cluster.

For the swift-storage charms, where currently one can only add devices but not
remove, the proposal is to support removing devices. This will entail
messaging the swift-proxy on the storage relation which an updated list of
devices and a new setting 'purge-missing-devices' to instruct the swift-proxy
to remove devices from the ring that are no longer configured. We will also
need to ensure that the device cache located on the swift-storage unit from
which we are removing a device is also updated to no longer include the
device since not doing so would block the device from being re-added in the
future. As an extension to this we should also extend the
swift-storage-relation-broken to support removing devices associated with that
unit/host from the rings and sycning these changes across the cluster.

[0] https://docs.openstack.org/swift/latest/ring_partpower.html
[1] https://docs.openstack.org/swift/latest/admin/objectstorage-ringbuilder.html#replica-counts

Alternatives
------------

Juju charm actions are another way to implement operational actions to be
performed on the cluster but do not necessarily fit all cases. Since ring
management is at the core of the existing charm (hook) code itself, the
proposal is to extend this code rather than move and rewrite it as an action.
However, there will likely be a need for some actions to be defined as
post-modification checks and cleanups which would be well suited to an
action and not directly depend on the charm ring manager.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  hopem

Gerrit Topic
------------

Use Gerrit topic swift-charm-extended-operations for all patches related to
this spec.

.. code-block:: bash

    git-review -t swift-charm-extended-operations

Work Items
----------

* add support for modifying partition-power
* add support for modifying replicas
* add support for removing devices
* add support for removing an entire swift-storage host

Repositories
------------

None

Documentation
-------------

All of the above additions will need to be properly documented in the charm
deployment guide.

Security
--------

None

Testing
-------

Each additional level of support will need very thorough testing against a
real Swift object deployed with the charms that contains data and is of a
reasonable scale. All code changes will be accompanied by unit tests and
where possible functional tests.

Dependencies
============

None

