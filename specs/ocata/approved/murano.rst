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

============
Murano Charm
============

To add a service to Openstack provide a catalogue of applications deployable
on Openstack.

Problem Description
===================

To provide UI and API which allows to compose and deploy composite
environments on the Application abstraction level and then manage their
lifecycle. The Service should be able to orchestrate complex circular dependent
cases in order to setup complete environments with many dependent applications
and services. However, the actual deployment itself will be done by the
existing software orchestration tools (such as Heat), while the Murano project
will become an integration point for various applications and services.

Proposed Change
===============

One new charm - Murano with corresponding tests and QA CI/setup.

The new Murano charm should include, as a minimum, the following features:

- Deployable in a highly available configuration
- Allow clients and services to interact using SSL encryption
- Charm progress displayed via workload status

Alternatives
------------

Jobs could scheduled manually via cron on each machine.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  unknown

Gerrit Topic
------------

Use Gerrit topic "murano" for all patches related to this spec.

.. code-block:: bash

    git-review -t murano

Work Items
----------

Provide Murano charm
++++++++++++++++++++

- Create skeleton charm layer based on OpenStack base layer and available
  interface layers to deploy Murano.
- Add support for upgrading Murano
- Add config option and accompanying support for upgrades via
  action-managed-upgrade.
- Add support for deploying Murano in a highly available configuration
- Add support for the Murano to display workload status
- Add support SSL endpoints
- Charm should have unit and functional tests.

Mojo specification deploying and testing Murano
+++++++++++++++++++++++++++++++++++++++++++++++

- Write Mojo spec for deploying murano in an HA configuration and testing
  creation of jobs.

Repositories
------------

A new git repository will be required for the Murano charm:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-murano

Documentation
-------------

The Murano charm should contain a README with instructions on deploying the
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
- Provide horizon interface layer
- Provide heat interface layer
- Provide hacluster interface layer
- Provide nrpe-external-master interface layer
- Provide OpenStack base layer with all common hook code that is not already
  covered by an interface layer.
- Provide OpenStack base layer with support for HA deployments
- Provide OpenStack base layer with support for SSL communication
- Provide OpenStack base layer with support for workload status
