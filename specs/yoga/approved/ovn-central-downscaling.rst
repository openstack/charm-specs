..
  Copyright 2022 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===========================================================
Enable clean removal of a ovn-central unit from the cluster
===========================================================

Add capability to ovn-central charm to gracefully leave Southbound and
Northbound clusters. This should be performed automatically in the process
of unit's teardown. In addition, there should be an action that enables
user to manually kick specified cluster member in case that the unit was not
able to gracefully leave the cluster on its removal.

It's important to keep in mind that each ovn-central unit is always part of two
clusters simultaneously, Northbound and Southbound. Any action in regards to
leaving or kicking of the unit must be performed on both of them.

Problem Description
===================

Currently there's no clean way to remove ovn-central unit from the cluster. As
described in `LP#1948680`_, removed units do not properly leave the Southbound
and Northbound clusters. Remaining active cluster members will still expect
them and their continued presence will affect fault tolerance of the whole
cluster. For example starting with a cluster of five ovn-central units and
removing two of them will cause the cluster to behave as a cluster of five
members with two faults, not a cluster of three members with no fault.


Proposed Change
===============

Part of the solution to this problem is to invoke "ovs-appctl cluster/leave"
command when the unit is in the process of shutting down. This will ensure
that remaining cluster members will reconfigure themselves and will no longer
expect the leaving unit to come back. These two commands should be executed
for the unit to leave both clusters::

    ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/leave OVN_Northbound
    ovs-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/leave OVN_Southbound


However there's also a possibility that departing unit won't be able to execute
"cluster/leave" command properly during the teardown so there should also be
an explicit action to kick "zombie" cluster members that were left behind after
their ovn-central unit disappeared.

Action: kick
------------

This action will enable user to kick specific members from the Southbound and
Northbound OVN clusters. It will require user to supply a short version of an
ID of the server that should be kicked from the cluster. Since the ovn-central
unit is member in two clusters, under two unique identities, this action will
accept following parameters:

* --sb-identity <ID> - ID of the server in Southbound cluster
* --nb-identity <ID> - ID of the server in Northbound cluster
* --i-really-mean-it - This option signifies that user is aware of destructive
  nature of this action

Example::

    juju run-action ovn-central/0 kick sb-identity=13fe nb-identity=77eb i-really-mean-it=true


Above example would translate to unit executing following commands::

    ovs-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/kick OVN_Southbound 13fe
    ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/kick OVN_Northbound 77eb

This action will be executable on any unit that's still participating in
the OVN cluster, it does not have to be leader and it only needs to be
executed once. The only problematic part is that user will have to provide
Southbound and Northbound IDs which are not really exposed via juju. To allow
user to correlate specific ovn-central unit with its cluster identities, we
propose a new action called "cluster-status"

Action: cluster-status
----------------------

Prerequisite for this action is to add two new data attributes to the peer
relation between ovn-central units implemented by `OVSDB Interface`_:

* sb-identity: Short version of the UUID that unit uses in Southbound cluster
* nb-identity: Short version of the UUID that unit uses in Northbound cluster

Each unit will publish its Northbound and Southbound ID via the peer relation.

The action itself will convey outputs of following commands with minor
formatting changes::

    ovs-appctl -t /var/run/ovn/ovnsb_db.ctl cluster/status OVN_Southbound
    ovs-appctl -t /var/run/ovn/ovnnb_db.ctl cluster/status OVN_Northbound

Example output of one of these commands looks something like this::

    f3be
    Name: OVN_Southbound
    Cluster ID: 77eb (77eb6ddc-6910-403b-b8da-ce3d90815183)
    Server ID: f3be (f3be898b-e8e4-4719-b344-902444227dd7)
    Address: ssl:10.5.1.88:6644
    Status: cluster member
    Role: follower
    Term: 8
    Leader: d33d
    Vote: d33d

    Election timer: 4000
    Log: [25, 25]
    Entries not yet committed: 0
    Entries not yet applied: 0
    Connections: ->9ea4 ->fbfc <-fbfc <-9ea4 <-d33d ->d33d
    Disconnections: 4
    Servers:
        d33d (d33d at ssl:10.5.0.92:6644) last msg 839 ms ago
        9ea4 (9ea4 at ssl:10.5.3.144:6644) last msg 81662711 ms ago
        fbfc (fbfc at ssl:10.5.1.86:6644) last msg 1221658 ms ago
        f3be (f3be at ssl:10.5.1.88:6644) (self)

The only proposed change in the following output would be in the "Servers"
section. There, we would drop information about "last msg" as it is not very
informative because cluster followers (non-leaders) don't keep active
heartbeats between each other. Instead of it, we would add information about
which unit the specific server belongs to.

Example::

    Servers:
        d33d (d33d at ssl:10.5.0.92:6644) ovn-central/0
        9ea4 (9ea4 at ssl:10.5.3.144:6644) ovn-central/1
        fbfc (fbfc at ssl:10.5.1.86:6644) ovn-central/2
        f3be (f3be at ssl:10.5.1.88:6644) UNKNOWN  # in case that this identity does not belong to any other related juju unit

With these information, user can effectively use action "kick" to remove
undesirable members from the cluster.

These features would be supported on Openstack Yoga (and later) on Ubuntu
22.04 LTS.

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Martin Kalcok <martin.kalcok@canonical.com>

Gerrit Topic
------------

Use Gerrit topic "ovn-central-downscaling" for all patches related to this
spec.

.. code-block:: bash

    git-review -t ovn-central-downscaling

Work Items
----------

* Implement automatic execution of "ovs-appctl cluster/leave" command when
  ovn-central unit is removed

* Implement announcing of Southbound and Northbound identity via
  `OVSDB Interface`_

* Implement "cluster-status" juju action in ovn-central charm

* Implement juju action "kick" that will enable removal of arbitrary cluster
  members from Southbound and/or Northbound clusters

Repositories
------------

* https://opendev.org/x/charm-ovn-central
* https://opendev.org/x/charm-interface-ovsdb.git

Documentation
-------------

README.md of the ovn-central charm will have to be updated to include
information about new juju actions.

Security
--------

This change does not pose any direct security risk. The action "kick" can be
dangerous when used incorrectly so it will include "--i-really-mean-it"
required flag, to signify to the user that it needs to be used carefully.

Testing
-------

Aside from unit tests, this change will have to be accompanied also with
functional tests that will verify that:

* removing unit will remove it from Southbound and Northbound clusters
* using juju action "kick" will remove the specified members from the clusters
* juju action "cluster-status" displays correct information

Dependencies
============

None

.. _LP#1948680: https://bugs.launchpad.net/charm-ovn-central/+bug/1948680
.. _OVSDB Interface: https://opendev.org/x/charm-interface-ovsdb/src/branch/master/src/lib/ovsdb.py
