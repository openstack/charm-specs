..
  Copyright 2019 Canonical UK

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
Masakari
===============================

Masakari provides an API and ENGINE which receive messages from MONITORS
(agents) on compute nodes to enable:

* instance evacuation on host failure (masakari-hostmonitor) - requires shared
  storage

* instance restart on certain instance errors (masakari-instancemonitor)

* openstack related process monitoring and restarting
  (masakari-processmonitor)

This spec is for two charms which will install the masarki api and engine, and
monitors respectively.

Problem Description
===================

If an openstack hypervisor fails, there is no method of automatic recovery.
Masakari provides a way of migrating instances from a failed host to a
functional host, when used in conjunction with a cluster manager.
There are similar issues with instance and process failures which masakari
addresses.

Proposed Change
===============

A solution using corosync and pacemaker on each hypervisor, along with
masakari-hostmonitor has been validated. Unfortunately a cluster where each node
runs the full cluster stack is only recommended up to around 16 nodes:

http://clusterlabs.org/pacemaker/doc/en-US/Pacemaker/1.1/html-single/Pacemaker_Remote/#_overview

The scale issue with this initial solution can be mitigated by breaking
compute nodes up into smaller clusters but this should be seen as a work
around.

They suggested approach to scale to a greater number of nodes is by using
pacemaker-remote on the hypervisors rather than the full cluster stack.
Unfortunately masakari-hostmonitors does not currently work with
pacemaker-remote due to the following bug:

https://bugs.launchpad.net/masakari-monitors/+bug/1728527

This work will involve the following new charms:

* masakari (provides api, engine and python-masakariclient)
  The masakari charm will have identity-service, amqp and shared-db relations

* masakari-monitors (optionally provides one or all of hostmonitor,
  instancemonitor, processmonitor)
  The masakari-monitors charm will be related to nova-compute via the juju-info
  relation and related to keystone via the identity-credentials relation. The
  monitors needs credentials for posting notifications to the masakari api
  service. When not using pacemaker_remote masakari-monitors will rely on the
  hacluster charm having been related to the nova-compute charm via juju-info.

* pacemaker-remote.
  This charm simple installs the pacemaker remote service and requires a
  relation with fully fledged cluster.


This work will involve the following new relations:

* hacluster <-> pacemaker-remote
  This relation will allow the pacemaker-remote charm to inform the hacluster
  charm of the location of remote nodes to be added to the cluster. It will
  also be used to expose the pacemaker-remotes stonith information and
  whether or not the pacemaker remote node should run resources. In the case
  of a masakri deployment the pacemaker-remote nodes should be set to
  not run resources.

The following charm updates:

* hacluster charm.
  The hacluster charm need to be able to support adding pacemaker-remote nodes
  to the cluster and also confgiuring resources such that if requested no
  resources are run on the remote nodes.

* hacluster charm.
  The hacluster charm need to be able to support creating maas stonith devices

Additional work:

* Fix masakari-hostmonitors to work with pacemaker remote
  As mentioned masakari-hostmonitors does not work wih pacemaker-remote at the
  moment. A patch will need to be written to fix this and ideally landed
   upstream.

* Write maas stonith driver.
  It is important that instances that are using shared storage are only running
  on one compute node at a time to avoid a split-brained cluster leading to
  two instances writting to the same storage simultaniously which would result
  in data corruption.

These charms will support TLS for API communications

Alternatives
------------

Although not as feature rich as maskari much of the functionality can be
achieved using pacemaker/corosync resource configuration.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gnuoy

Secondary assignee:
  admcleod

Gerrit Topic
------------

Use Gerrit topic "masakari" or "masakari-monitors" as appropriate for all
patches related to this spec.

.. code-block:: bash

    git-review -t masakari
    git-review -t masakari-monitors


Work Items
----------

* Masakari charm cleanup
* Add domain id information to identity-credentials relation. The keystone charm
and the identity-credentials interface will both need updating.
* Masakari-monitors charm cleanup
* Add pacemaker-remote charm
* Add pacemaker-remote interface
* Add functional and unit tests
* Mojo specs for full stack functional testing.
* Write patch to fix pacemaker-remote support in masakari-hostmonitors
* Write Maas stonith plugin
* Extend hacluster charm to support registering pacemaker_remote nodes.
* Extend hacluster charm to support only running resources on a subset
  of available nodes.
* Extend hacluster charm to support creating maas stonith devices.
* Write deployment guide instructions.
* Add new charms and interfaces to OpenStack gerrit.


Repositories
------------

git://git.openstack.org/openstack/charm-masakari
git://git.openstack.org/openstack/charm-masakari-monitors

Documentation
-------------

Both charms will contain README.md files with instructions

Security
--------

We will have credentials for the cloud stored on the compute node.  Dropping
from the guest to the host in this case could allow a user to compromise the
cloud by signally the masakari api about one or more false compute node
failures. Keystone credentials which are used by the placement api are
already stored on the compute node so this does not increase the attack
surface but is worth mentioning for completeness.

We will need to enable a certificate relation in the nova compute host to
facilitate the use of a vault charm to enable masakari ssl functionality.


Testing
-------

Code changes will be covered by unit tests; functional testing will be done
using a combination of zaza and Mojo specification.

Dependencies
============

- Requires cluster management such as corosync or pacemaker. At the very least,
  hacluster charm is required

- Shared storage is required

- Some administrative intervention will be required after a host failure.
