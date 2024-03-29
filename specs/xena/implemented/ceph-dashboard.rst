..
  Copyright 2020 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

============================
Enablement of Ceph Dashboard
============================

Problem Description
===================

The ceph-mon charm provides the cluster monitor daemon for the Ceph
distributed file system. It currently does not support enablement of
the Ceph Manager built-in Dashboard. It is a web-based Ceph management
and monitoring application through which you can inspect and administer
various aspects and resources within the cluster. It is implemented as
a Ceph Manager Daemon module.


Proposed Change
===============

The proposed solution is to make the dashboard available and default enabled
in a new subordinate charm to ceph-mon. SSL endpoint termination would be done
using the certificates relation or by providing the ssl_cert, ssl_key and
ssl_ca configuration options. User management would be done with the use of
charm actions.

The upstream documentation of Ceph mentions Dashboard is supported,
(https://docs.ceph.com/en/latest/mgr/dashboard/). The new feature would be
supported on Ceph Octopus, OpenStack Victoria (and later) on Ubuntu 20.04 LTS.

The subordinate charm will enable access to the dashboard via the hostnames of
the mon units. However, the hostname may not map to the desired network space,
the operator may not be able to resolve the juju unit hostnames and the failure
of a single mon unit may make the dashboard inaccessable if the operator was
accessing the dashboard via that mon. To resolve these issues
the dashboard charm will support a relation to an OpenStack loadbalancer
application (as yet unwritten) which will manage an haproxy instance and a
vip via the hacluster subordinate. If this relation is present the dashboard
standby units will be configured to return a 500 enabling haproxy to determine
the active dashboard unit. Both the loadbalancer and the dashboard charm
will support SSL endpoint termination via a relation to vault or via
configuration options. The delivery of the dashboard subordinate should be the
initial focus of this work with the loadbalacer following soon after.

Alternatives
------------

Grafana offers some visualization possibilities, but it is limited to view
only experience.

Out of Scope
------------

.. list-table:: Features
   :widths: 25 25
   :header-rows: 1

   * - Feature
     - In Scope
   * - Multi-User and Role Management
     - ✓
   * - Single Sign-On (SSO)
     - X
   * - SSL/TLS support
     - ✓
   * - Auditing
     - ✓
   * - Embedded Grafana Dashboards
     - ✓
   * - Monitoring (Prometheus & Alertmanager)
     - X
   * - iSCSI
     - X
   * - RBD
     - ✓
   * - RBD mirroring
     - X
   * - CephFS
     - X
   * - Object Gateway
     - ✓
   * - NFS
     - X


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Liam Young <liam.young@canonical.com>

Gerrit Topic
------------

Use Gerrit topic "ceph-dashboard" for all patches related to this spec.

.. code-block:: bash

    git-review -t ceph-dashboard

Work Items
----------

* Implement a new operator framework subordinate charm, ceph-dashboard

* Add a charm option for toggling ceph mgr dashboard

* Add a relation that implements SSL endpoint configuration

* Add actions to manage user accounts

  * Create admin user

* write unit tests

* extend current functional tests using the zaza test framework (a functional
  test should include deploying ceph-mon with ceph-dashboard added, without
  user management actions)

* Create generic service loadbalancer charm.

* Add a relation to a loadbalancer service which will provide a HA vip
  for accessing the dashboard.


Timeline
--------

The goal is to implement this change in the OpenStack Charms 21.10 release.

Repositories
------------

During initial development the code will be managed here:

https://github.com/openstack-charmers/charm-ceph-dashboard

When ready the charm will be managed via OpenDev gerrit.

https://opendev.org/openstack/charm-ceph-dashboard



Documentation
-------------

The charm should contain documented options:

* Create charm options

* Create charm actions

* Create charm relations

* Create charm README

* Update of the official charm deployment guide: [0]

[0]: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/


Security
--------

The Dashboard endpoint has to be TLS-terminated, therefore, an option to
provide a CA certificate is required.

Testing
-------

Code changes will be covered by unit tests.
Functional tests would require ceph OSD hardware or software emulation.
This is possible through Ceph cluster deployment. The deployment of the
ceph cluster would be independent from the OpenStack bundle.


Dependencies
============

- This charm will support OpenStack Victoria and
  Ubuntu 20.04 Focal as its baseline
- A new project will need to be created based on the OpenStack Project
  `Creator's Guide <https://docs.openstack.org/infra/manual/creators.html>`
