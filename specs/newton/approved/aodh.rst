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

==========
Aodh Charm
==========

Aodh (alarms and notifications based on metrics) was split from Ceilometer
during the Liberty cycle and equivalent function was removed from
Ceilometer during the Mitaka cycle.

We need to charm Aodh to allow OpenStack Operators to deploy and use
alarming and notification services as part of an OpenStack cloud.

Problem Description
===================

Aodh provides a method generating alarms and notifications based on
metrics.

Proposed Change
===============

Aodh will need to undergo MIR review for main inclusion; this process
should include stripping of all debconf/dbconfig related code from the
packaging.

The new Aodh charm should include, as a minimum, the following features:

- Deployable in a highly available configuration
- Allow clients and services to interact using SSL encryption
- Charm progress displayed via workload status

Alternatives
------------

Manually review metrics and trigger alarms.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  jamespage

Gerrit Topic
------------

Use Gerrit topic "aodh" for all patches related to this spec.

.. code-block:: bash

    git-review -t aodh

Work Items
----------

Provide fully supported packages for Ubuntu
===========================================

- Package updates for Aodh to strip all debconf/dbconfig related code from
  the packaging.
- MIR review for Aodh - Evaluate package and source according to
  https://wiki.ubuntu.com/MainInclusionProcess, open corresponding bug, work
  with Ubuntu MIR team and make any other necessary package changes to get
  package into main.

Provide Aodh charm
==================

- Create skeleton charm layer based on OpenStack base layer and available
  interface layers to deploy Aodh.
- Add support for upgrading Aodh
- Add config option and accompanying support for upgrades via
  action-managed-upgrade.
- Add support for deploying Aodh in a highly available configuration
- Add support for the Aodh to display workload status
- Add support SSL endpoints
- Charm should have unit and functional tests.

Mojo specification deploying and testing Aodh
=============================================

- Write Mojo spec for deploying Mojo in an HA configuration and testing
  automatic and manual creation of DNS records.

Repositories
------------

A new git repository will be required for the Aodh charm:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-aodh

Documentation
-------------

The Aodh charm should contain a README with instructions on deploying the
charm. A blog post is optional but would be a useful addition.

Security
--------

No additional security concerns.

Testing
-------

Code changes will be covered by unit tests; functional testing will be done
using a combination of Amulet, Bundle tester and Mojo specification.

Dependencies
============

- Provide mongodb interface
- Provide hacluster interface layer
- Provide nrpe-external-master interface layer
- Provide OpenStack base layer with all common hook code that is not already
  covered by an interface layer.
- Provide OpenStack base layer with support for HA deployments
- Provide OpenStack base layer with support for SSL communication
- Provide OpenStack base layer with support for workload status
