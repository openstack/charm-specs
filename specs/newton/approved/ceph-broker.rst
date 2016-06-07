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

Connect an existing Ceph to a juju deployed Openstack

Problem Description
===================

Some customers have asked how to connect a Juju deployed Ceph to an existing
OpenStack cluster that was deployed with home grown or someone else’s tooling.
This causes a problem in that it is not possible to relate Ceph to the
OpenStack cluster.

Proposed Change
===============

A new charm will be created that acts a a broker between an existing Ceph deployment,
and a Juju deployed OpenStack Cloud; The charm will provide the same relations as
the existing ceph-mon charm.


Alternatives
------------

The alternative is manually connecting the Ceph and OpenStack together.  This is
fine for some customers but not acceptable for bootstack.  This kind of manual
configuration isn't particularly manageable.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ChrisMacNaughton

Gerrit Topic
------------

    git-review -t ceph-broker

Work Items
----------

 * Decide on which relations the OpenStack charm requires
 * Expose all relations needed by way of config.yaml options.
 * For every relation that OpenStack expects just return the config.yaml
 values.

Repositories
------------

Yes a new repo will be needed for this.
https://github.com/openstack/charm-ceph-broker

Documentation
-------------

Documentation will be added to the README.md as part of the normal workflow.

Security
--------

No additional security concerns.

Testing
-------

* Deploy OpenStack using juju.
* Deploy Ceph using juju.
* Deploy the Ceph-broker to a lxd container or a vm after filling out the
config.yaml
* Relate Ceph-broker to OpenStack and verify that OpenStack can talk to Ceph
* Mojo bundle tests will be used to show this works functionally.

Dependencies
============
None
