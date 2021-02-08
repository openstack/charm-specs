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

======
Magnum
======

Magnum is an OpenStack API service, developed by the OpenStack Containers team,
making container orchestration engines (COEs) such as Docker Swarm, Kubernetes,
and Apache Mesos available as first class resources in OpenStack.

It uses Heat to orchestrate an OpenStack image, which contains the
COE (for example: Docker and Kubernetes), and it runs that image in either
virtual machines or bare metal in a cluster configuration.

Also, Magnum offers complete life-cycle management of COEs in an OpenStack
environment, integrated with other OpenStack services. This allows a seamless
experience for OpenStack users who wish to run containers in an OpenStack
environment.

Problem Description
===================

There is currently no charm that lets you easily deploy Magnum as part of
Charmed OpenStack.

Proposed Change
===============

Implement a set of reactive charms that enables the deployment of all required
Magnum components. Changes will also be made to existing charms to integrate
with the new Magnum charms.

Baseline supported OpenStack version will be Ussuri.

Alternatives
------------

None

Implementation
==============

The implementation consists of the following new charms:

- magnum

  - Provides the Magnum API. OpenStack administrators and Openstack Dashboard
    will call into this API to provision and manage Kubernetes clusters (or
    any other COE).

  - Requires relations to AMQP, certificates, database and Keystone.

- magnum-dashboard

  - Create a new subordinate charm for Openstack Dashboard

  - Implements the necessary changes to enable Magnum UI in Horizon

The implementation also requires changes to the following charms:

- keystone

  - Add Magnum to the list of valid services.

Assignee(s)
-----------

Primary assignee:
  oprinmarius

Gerrit Topic
------------

Use Gerrit topic "magnum-charm" for all patches related to this spec.

.. code-block:: bash

    git-review -t magnum-charm


Work Items
----------

The implementation section describes the working items.

Repositories
------------

- charm-magnum

- charm-magnum-dashboard

- charm-keystone

Documentation
-------------

Each charm will contain a README with instructions on deploying the charm.
In addition, an appendix will be added to the deployment guide which will
outline various deployment scenarios, hardware and networking requirements
and config options.

Security
--------

- ``magnum``

  - Requires keystone, database and rabbitmq credentials which will be
    stored in ``magnum.conf``

  - The API endpoints for Magnum will optionally be accessible via TLS,
    either by creating a relation to Vault / easy-rsa, or by manually
    supplying the TLS key/cert via config


Testing
-------

Code written or changed will be covered by unit tests; functional testing will
be implemented using the ``Zaza`` framework.

Dependencies
============

None

