..
  Copyright 2017 Canonical LTD

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
Service Restart Control In Charms
===============================

Openstack charms continuously respond to hook events from their peers
related applications which frequently result in configuration
changes and subsequent service restarts. This is all fine until these
applications are deployed at large scale and having these services restart
simultaneously can cause (a) service outages and (b) excessive load on
external applications e.g. databases or rabbitmq servers. In order to
mitigate these effects we would like to introduce the ability for charms
to apply controllable patterns to how they restart their services.

Problem Description
===================

An example scenario where this sort of behaviour becomes a problem is where
we have a large number, say 1000, of nova-compute units all connected to the
same rabbitmq server. If we make a config change e.g. enable debug logging
on that application this will result in a restart all nova-* services on
every compute host in tandem which will in turn generate a large spike of
load on the rabbit server as well as making all compute operations block
until these services are back up. This could also clearly have other
knock-on effects such as impacting other applications that depend on
rabbitmq.

There are a number of ways that we could approach solving this problem but
for this proposal we choose simplicity by attempting to use all information
already available to an application unit combined with some user config to
allow units to decide how best to perform these actions.

Every unit of an application already has access to some information that
describes itself with respect to its environment e.g. every unit has a unique
id and some applications have a peer relation that gives them information
about their neighbours. Using this information coupled with some extra
config options on the charm to vary timing we could provide the operator
the ability to control service restarts across units using nothing more
than basic mathematics and no juju api calls.

For example, let's say an application unit knows it has id 215 and the user
has provided two options via config; a modulo value of 2 and an offset of
10. We could then do the following:

.. code:: python

    time.sleep((215 % 2) * 10)

which, when applied to all units, would result in 50% of the cluster
restarting its services 10 seconds after the rest. This should hopefully
alleviate some of the pressure resulting from cluster-wide synchronous
restarts, ensuring that part of the cluster is always responsive and
making restarts happen quicker.

As mentioned above we will require two new config options to any charm for
which this logic is supported:

* service-restart-offset (default to 10)
* service-restart-modulo (default to 1 so that default behaviour is same as
                          before)

The restart logic will skip for any charms not implementing these options.

Over time some units may be deleted from and added back to the cluster
resulting in non-contiguous unit ids. While for applications deployed at
large scale this is unlikely to be significantly impactful, since subsequent
adds and deletes will cancel each other out, it could nevertheless be a
problem so we will check for the existance of a peer relation on the
application we are running and, if one exists, use the info in that relation
to normalise unit ids prior to calculating delays.

Lastly, we must consider how to behave when the charm is being used to upgrade
Openstack services whether directly using config ("big bang") or using actions
defined on a charm. For the case where all services are upgraded at once we
will leave it to the operator to set/unset the offset parameters. For the case
where actions are being used, and likely only a subset of units are being
upgraded at once, we will ignore the control settings i.e. delays will not
be used.

Proposed Change
===============

To implement this change we will extend the restart_on_change() decorator
implemented across the openstack charms so that when services are stop/started
or restarted they will include a time.sleep(delay) where delay is
calculated from unit id combined with two new config options;
service-restart-offset and service-restart-modulo. This calculation will be
done in a new function that will be implemented in contrib.openstack the
output of which will be passed into the restart_on_changed() decorator.

Since a decorator is used we do not need to worry about multiple restarts of
the same service. We do, however, need to consider how apply offsets when
stop/start and restarts are performed manually as is the case in the action
managed upgrades handler.

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  hopem

Gerrit Topic
------------

Use Gerrit topic "controlled-service-restarts" for all patches related to
this spec.

.. code-block:: bash

    git-review -t controlled-service-restarts

Work Items
----------

* implement changes to charmhelpers
* sync into openstack charms and add new config opts

Repositories
------------

None

Documentation
-------------

These new settings will be properly documented in the charm config.yaml as
well as in the charm deployment guide.

Security
--------

None

Testing
-------

Unit tests will be provided in charm-helpers and functional tests will be
updated to include config that enables this feature. Scale testing to prove
effectiveness and determine optimal defaults will also be required.

Dependencies
============

None
