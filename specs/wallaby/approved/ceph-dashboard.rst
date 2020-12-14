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


Out of Scope
============

The following items should be covered in any future work:
 - integration with current LMA stack solution
 - SAML integration
 - RadosGW integration
 - Auditing
 - iSCSI Management
 - verify compatibility with previous releases, OpenStack Mimic+ (Stein)
 - VIP and hacluster relation

Alternatives
------------

Grafana offers some visualization possibilities, but it is limited to view
only experience.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Gabor Meszaros <gabor.meszaros@canonical.com>

Gerrit Topic
------------

Use Gerrit topic "ceph-dashboard" for all patches related to this spec.

.. code-block:: bash

    git-review -t ceph-dashboard

Work Items
----------

* Implement a new reactive subordinate charm, ceph-dashboard

* Add a charm option for toggling ceph mgr dashboard

* Add a relation that implements SSL endpoint configuration

* Add actions to manage user accounts

  * Create admin user

* write unit tests

* extend current functional tests using the zaza test framework (a functional
  test should include deploying ceph-mon with ceph-dashboard added, without
  user management actions)


Timeline
--------

The goal is to implement this change in the OpenStack Charms 21.04 release.
The freeze date for this release is 15th Mar 2021, for a release on 9th
Apr (see release schedule). This change should be proposed for merging at
least two weeks ahead of freeze, so ideally submitted by 26th Feb 2021.

Repositories
------------

Changes will be pushed to a new ceph-dashboard charm repository:

https://git.openstack.org/openstack/charm-ceph-dashboard


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
