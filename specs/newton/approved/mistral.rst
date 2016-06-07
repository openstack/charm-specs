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

=====================
Mistral Charm Support
=====================

To add a service to Openstack to set up and manage on-schedule jobs for
multiple machine.

Problem Description
===================

As a cloud administrator I have maintenance jobs that I’d like to be run
against the cloud. I want to ble to set up and manage on-schedule jobs for
multiple machines. I’d like a single point of control over their schedule. 

As a devops engineer I’d like to be able to specify workflows needed for
deploying environments consisting of multiple VMs and applications. 

As a business process enforcer I’d like to be able to make a request to run a
complex multi-step business process and have it be fault-tolerant so that if
the execution crashes at some point on one node then another active node of the
system can automatically take on and continue from the exact same point where
it stopped. 

As a data analyst I need a tool for data crawling. Eg I’d like to be able to
express as a graph the related tasks I need in order to prepare a financial
report.

Proposed Change
===============

Mistral will need to undergo MIR review for main inclusion; this process
should include stripping of all debconf/dbconfig related code from the
packaging.

One new charm - Mistral with corresponding tests and QA CI/setup.

The new Mistral charm should include, as a minimum, the following features:

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

Use Gerrit topic "<topic_name>" for all patches related to this spec.

.. code-block:: bash

    git-review -t mistral

Work Items
----------

Provide fully supported packages for Ubuntu
===========================================

- Package updates for Mistral to strip all debconf/dbconfig related code from
  the packaging.
- MIR review for Mistral - Evaluate package and source according to
  https://wiki.ubuntu.com/MainInclusionProcess, open corresponding bug, work
  with Ubuntu MIR team and make any other necessary package changes to get
  package into main.

Provide Mistral charm
========================

- Create skeleton charm layer based on OpenStack base layer and available
  interface layers to deploy Mistral.
- Add support for upgrading Mistral
- Add config option and accompanying support for upgrades via
  action-managed-upgrade.
- Add support for deploying Mistral in a highly available configuration
- Add support for the Mistral to display workload status
- Add support SSL endpoints
- Charm should have unit and functional tests.

Mojo specification deploying and testing Mistral
================================================

- Write Mojo spec for deploying Mojo in an HA configuration and testing
  creation of jobs.

Repositories
------------

git@github.com:openstack-charmers/charm-mistral.git

Documentation
-------------

The Mistral charm should contain a README with instructions on deploying the 
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
