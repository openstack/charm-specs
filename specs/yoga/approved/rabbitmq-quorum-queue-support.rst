..
  Copyright 2021, Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

======================================
Add support for managing Quorum Queues
======================================

The RabbitMQ project is moving away from classic queues n favour of quorum
queues (see `4_0_deprecations`_). Quorum queues provide greater data safety
and are based on the widely adopted `Raft consensus algorithm`_

OpenStack does not currently support creating quorum queues but there is
a change up for review to add support for them `Oslo Messaging Quorum Queues`.

WARNING: There is not complete feature parity between classic queues and
quorum queues. For example quorum queues do not support message TTLs which
maybe an issue with notifications in OpenStack queues which often build up.
See `_feature_comparison` for more details.

Problem Description
===================

Quorum queues are available with RabbitMQ 3.8.0 or later (eg Focal). A
deployment using the existing rabbitmq-server charm on focal will already
support the creation of quorum queues. The replication of the queues needs
to be managed externally in the event that rabbitmq server cluster is
expanded or shrunk. This spec covers the features needed to manage
quorum queues throughout the rabbit clusters life-cycle. For example if a
rabbitmq-server unit is lost and it was hosting a mirror of a quorum
queue then a new mirror on another unit should be added.

NOTE: When a client connects to rabbitmq-server via the amqp relation they
declare the vhost that they want access to. The rabbitmq-server charm ensures
the vhost exists and provides the client with credentials to use the vhost. It
is up to the client to create any queues it needs and it is at the point that
the queue is created that its type is defined.

Proposed Change
===============

Add functionality to the peer joined, departed and broken hooks to rebalance
queues in the event of a peer being added or lost. These hooks will most
likely utilise the add_member, delete_member, grow and shrink options
to the rabbitmq-queues command.

Charm actions to manually add or remove members from a quorum queue should
also be added to aid with pro-active maintenance of the cluster like taking
a unit down for maintenance.

Alternatives
------------

Classic queues are still supported so an alternative would be to continue
to use them in all cases.

Implementation
==============

Assignee(s)
-----------

TBD

Gerrit Topic
------------

Use Gerrit topic "quorum_queues" for all patches related to this spec.

.. code-block:: bash

    git-review -t quorum_queues

Work Items
----------

* Add functions to rabbit utils to identify all quorum queues across all
  vhosts.
* Add function to rabbit utils to report which nodes are hosting a queue.
* Extend peer departed hook to identify which queues have a mirror hosted on
  the leaving node and for each queue add a mirror on another node if possible.
* Report queues with insufficient mirrors in charms workload status.
* Add actions to allow queues mirrors to be moved.

Repositories
------------

No new repositories needed

Documentation
-------------

Actions will need to be documented.

Security
--------

N/A

Testing
-------

* Add functional test to create quorum queues and check there are sufficient
  replicas after cluster is shrunk and expanded.

Dependencies
============

- For OpenStack to utilise quorum queues `Oslo Messaging Quorum Queues` needs
  to land and each OpenStack client charm will need to grow support for the
  `rabbit_quorum_queue` option.

.. LINKS
.. _Maintainer comment: https://github.com/rabbitmq/rabbitmq-server/discussions/2950#discussioncomment-563361
.. _4_0_deprecations: https://blog.rabbitmq.com/posts/2021/08/4.0-deprecation-announcements/
.. _Raft consensus algorithm: https://raft.github.io/
.. _Oslo Messaging Quorum Queues: https://review.opendev.org/c/openstack/oslo.messaging/+/807826
.. _feature_comparison: https://www.rabbitmq.com/quorum-queues.html#feature-comparison
