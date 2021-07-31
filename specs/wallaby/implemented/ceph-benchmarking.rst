..
  Copyright 2020, Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=================
Ceph Benchmarking
=================

Proposing a new charm, ceph-benchmarking, and additions to the Zaza test
framework, in order to provide a simple repeatable method for testing ceph
performance.

This new charm will allow for A/B testing. Performance can be compared between
a deployment with a feature turned on and a deployment with the feature turned
off in order to make informed choices on feature usage.

`Benchmark Ceph Cluster Performance`_ will be used as template for the various
tests.


Problem Description
===================

It is often unclear what performance one should expect from a software stack
like Ceph. For example, it is difficult to give performance expectations in
absolute terms (e.g. IOPS, throuput, etc), because performance is hardware and
technology dependent. It may also be difficult to determine which suite of
features to use and what their relative cost / benefit quotient might be.

The proposed changes will allow for acquiring relative performance (IOPs) on a
given set of hardware that can be compared with various features turned on or
off. Testing can be performed on hardware prior to a production deployment in
order to give a before and after perspective on how the Ceph cluster performs
in order to assist in bottleneck detection.


Proposed Change
===============

* ceph-benchmarking charm

  A charm with several actions to run Ceph performance benchmarking tests and
  gather their results. (e.g. 'rados bench', 'rbd bench', 'fio', 'swift bench')
  The new charm will utilised the Ops Framework and leverage ops_openstack code
  base.

  - Proposed actions and parameters

    * net-iperf

      - IP(s) of Ceph cluster

    * fio-disk

      - disk(s)
      - readwrite
      - blocksize
      - iodepth

    * rados-bench

      - pool
      - duration
      - task (write, random, sequential)

    * rbd-bench

      - pool
      - image

    * fio-ceph

      - readwrite
      - blocksize
      - iodepth

    * swift-bench


* Additions to the Zaza test framework

  The Zaza test framework uses libjuju under the hood and allows for standing
  up deployments from bundles and executing suites of tests against them. Test
  targets for each action on the new charm will be added allowing for a suite
  of Ceph performance tests to be repeated as often as desired.


Alternatives
------------

All of the tests in `Benchmark Ceph Cluster Performance`_ can be run manually.
The goal is to make this process more efficient.


Implementation
==============

Assignee(s)
-----------

Primary assignee: David Ames (thedac)


Gerrit Topic
------------

Use Gerrit topic "ceph-benchmarking" for all patches related to this spec.

.. code-block:: none

   git-review -t ceph-benchmarking

Work Items
----------

* ceph-benchmarking charm


* Zaza targets


Repositories
------------

The ceph-benchmarking charm will initially reside in the ``openstack-charmers``
namespace and may eventually move to the ``openstack`` namespace:

https://github.com/openstack-charmers/charm-ceph-benchmarking

Additions to Zaza will be added to the zaza-openstack-tests repository:

https://github.com/openstack-charmers/zaza-openstack-tests


Documentation
-------------

Documentation will be written for the usage of ceph-benchmarking and related
Zaza tests. The actions for the charm will be documented in the charm's README.
The usage of the actions with or without Zaza will be an appendix to the
charm-guide and / or the deployment-guide.


Security
--------

The various tests may leave behind test detritus. Cleanup will be attempted
where possible, but the operator will need to take responsibility for any
security implications.


Testing
-------

The proposed changes are the tests themselves. The Zaza targets will provide
functional testing for the ceph-benchmarking charm.


Dependencies
============

- `Benchmark Ceph Cluster Performance`_ is the template for the various tests.


.. LINKS
.. _Benchmark Ceph Cluster Performance: https://tracker.ceph.com/projects/ceph/wiki/Benchmark_Ceph_Cluster_Performance
