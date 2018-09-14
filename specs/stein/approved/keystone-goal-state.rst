..
  Copyright 2018 Canonical Ltd.

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
Keystone Charm Goal State Support
=================================

Charms install and configure a piece of software on a unit in a model.  How the
software should be set up is determined by a set of information sources.

- Unit operating environment

- Application level configuration options

  - Set in the model and available to the charm at a early stage

- Subordinate relations to charms installed on the same unit

- Peer/Cluster relation to other units of same application in same model

- Relations to units of other applications in the same model

- Cross-model relations

In its simplest form a charm’s sole goal is to as quickly as possible install a
piece of software, write its configuration and fire it up enabling the end user
to make use of the software.

Problem Description
===================

As the deployment progresses the charm learns more about its broader
environment in pace with its `relation-joined` and `relation-changed` hooks
being fired.  Some of these relations may fundamentally change how the software
should be set up or behave.  Which dependencies and requirements a piece of
software needs met before it can be brought to a operational state also relies
heavily on which relations the charm has.

At present you may find that by the time a deployment is complete a charm has
reconfigured and restarted it’s services several times, each time it may also
have notified it’s peers and relations to do the same.  These activities
prolong the deployment time, and causes race conditions and transient errors on
depending services.

Proposed Change
===============

To address some of these issues Juju has introduced a `goal-state` command to
the hook environment.  This can be used by a charm to get a upfront view of
what Juju is going to deploy.

Scope
-----

- Introduce `goal-state` support in Keystone charm

- When available, make use of the early view of deployment to streamline charm
  operation

Limitations
-----------

- The feature has dependency on support from Juju.  At the time of this writing
  this feature is only supported by Juju 2.4.

  - When charm is deployed with versions of Juju without goal state support it
    falls back to inferring information about the deployment from configuration
    options and as relations appear.

- Goal state gives us a peek into a future, the reality of that future can
  change beneath us if modifications are applied to the model before deployment
  completes.

  - In the event of goal state changing mid-deployment, the benefits of the
    goal state optimizations may be minimized or in worst case make deploy time
    longer.

Alternatives
------------

Some of these issues can be alleviated by introducing configuration options
that give the charm a hint of expected scale.  But this approach is static and
contradicts the dynamic use and ease of scale out properties of Juju.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fnordahl

Gerrit Topic
------------

Use Gerrit topic "goal-state" for all patches related to this spec.

.. code-block:: bash

    git-review -t goal-state

Work Items
----------

- Write test charm and associated test specs to exercise and monitor how
  `identity-*` relations are exercised by the keystone charm in complex
  deployment scenarios.

- Delay initial run of `identity-admin`, `identity-service` and
  `identity-credentials` relations until:

  - all keystone units have joined and completed the charms peer/cluster
    relation

  - `shared-db` relation is ready and settled

  - If HA, make sure it is really up and ready

- Limit the number of times `identity-admin`, `identity-service` and
  `identity-credentials` relations are fired after initial run

- Limit the number of times credentials are added/updated to the database

Repositories
------------

No new repositories required.

Documentation
-------------

This is mainly a under the hood change and documentation will be added as part
of the code in docstrings.

Security
--------

No changes affecting security.

Testing
-------

Unit tests will be written alongside any code changes.

Functional tests will be expanded or written as necessary.

A test charm and associated test specs will be written to exercise and monitor
how `identity-*` relations are exercised by the keystone charm in complex
deployment scenarios.

Comparison of deployment times prior to and after the proposed change will be
documented as part of the review process.

Dependencies
============

- Juju 2.4 is required for goal-state support, charm will fall back to current
  behaviour when this is not available.
