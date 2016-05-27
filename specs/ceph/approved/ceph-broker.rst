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
Ceph Broker
===============================

Connect Ceph to existing Openstack’s

Problem Description
===================

Some customers have asked how to connect a Juju deployed Ceph to an existing
OpenStack cluster that was deployed with home grown or someone else’s tooling.
This causes a problem in that it is not possible to relate Ceph to the
OpenStack cluster.

Proposed Change
===============

The idea here is to create a charm that can provide the information that Openstack
needs to connect to the existing Ceph cluster.


Alternatives
------------

The alternative is manually connecting the Ceph and OpenStack together.  This is
fine for some customers but not acceptable for bootstack.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ChrisMacNaughton

Gerrit Topic
------------

Use Gerrit topic "<topic_name>" for all patches related to this spec.

.. code-block:: bash

    git-review -t ceph-broker

Work Items
----------

Work items or tasks -- break the feature up into the things that need to be
done to implement it. Those parts might end up being done by different people,
but we're mostly trying to understand the timeline for implementation.

Repositories
------------

Will any new git repositories need to be created?

Documentation
-------------

Will this require a documentation change?  If so, which documents?
Will it impact developer workflow?  Will additional communication need
to be made?

Security
--------

Does this introduce any additional security risks, or are there
security-related considerations which should be discussed?

Testing
-------

What tests will be available or need to be constructed in order to
validate this?  Unit/functional tests, development
environments/servers, etc.

Dependencies
============
- charm-layering
