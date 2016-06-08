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

===========
Trove Charm
===========

To add a service to Openstack to provide on-demand databases.

Problem Description
===================

As a cloud user I need be able to deploy a database that is scalable and
reliable quickly and easily without the burden of handling complex
administrative tasks.

Proposed Change
===============

Trove will need to undergo MIR review for main inclusion; this process
should include stripping of all debconf/dbconfig related code from the
packaging.

One new charm - Trove with corresponding tests and QA CI/setup.

The new Trove charm should include, as a minimum, the following features:

- Deployable in a highly available configuration
- Allow clients and services to interact using SSL encryption
- Charm progress displayed via workload status

Alternatives
------------

.. code-block:: bash

    juju deploy {mysql,percona-cluster,cassandra,etc}

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  unknown

Gerrit Topic
------------

Use Gerrit topic "trove" for all patches related to this spec.

.. code-block:: bash

    git-review -t trove

Work Items
----------

Provide fully supported packages for Ubuntu
+++++++++++++++++++++++++++++++++++++++++++

- Package updates for Trove to strip all debconf/dbconfig related code from
  the packaging.
- MIR review for Trove - Evaluate package and source according to
  https://wiki.ubuntu.com/MainInclusionProcess, open corresponding bug, work
  with Ubuntu MIR team and make any other necessary package changes to get
  package into main.

Provide Trove charm
+++++++++++++++++++

- Create skeleton charm layer based on OpenStack base layer and available
  interface layers to deploy Trove.
- Add support for upgrading Trove
- Add config option and accompanying support for upgrades via
  action-managed-upgrade.
- Add support for deploying Trove in a highly available configuration
- Add support for the Trove to display workload status
- Add support SSL endpoints
- Charm should have unit and functional tests.

Mojo specification deploying and testing Trove
++++++++++++++++++++++++++++++++++++++++++++++

- Write Mojo spec for deploying Trove in an HA configuration and testing
  creation of databases.

Repositories
------------

A new git repository will be required for the Trove charm:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-trove

Documentation
-------------

The Trove charm should contain a README with instructions on deploying the
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

- Provide rabbitmq interface layer
- Provide mysql-shared interface layer
- Provide pgsql interface layer
- Provide keystone interface layer
- Provide hacluster interface layer
- Provide nrpe-external-master interface layer
- Provide OpenStack base layer with all common hook code that is not already
  covered by an interface layer.
- Provide OpenStack base layer with support for HA deployments
- Provide OpenStack base layer with support for SSL communication
- Provide OpenStack base layer with support for workload status
