..
  Copyright 2019 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=======
MySQL 8
=======

Introduce charmed and enterprise grade MySQL 8 + InnoDB + MySQL Router as a
database solution for OpenStack models with the intention of making it the
default as of Ubuntu 20.04.

Problem Description
===================

Percona XtraDB Cluster is in Universe, has failed a Main Inclusion Request
process, and will not be carried into the next LTS relative to Charmed
OpenStack deployments.


Proposed Change
===============

These will be wholly new charms, not based in any way on existing MySQL or
Percona-Cluster charms, and without a charm upgrade path.

Instead of in-place upgrades a migration solution will be provided to allow
existing Ubuntu 18.04 LTS deployments using the percona-cluster charm to
migrate databases directly to an Ubuntu 20.04 LTS deployment based on
mysql-innodb-cluster. Full details will be provided as an update to this spec.

The charms will be promulgated to the following locations:

- cs:mysql-innodb-cluster
- cs:mysql-router


What versions of the operating system are affected or required?
19.10 (Eoan) and 20.04

What versions of OpenStack are affected or required?
Train and above

What version of Juju is required?
Juju 2.7+


Alternatives
------------

The alternative is to bring percona-cluster into main for 20.04.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  thedac


Gerrit Topic
------------

Use Gerrit topic "mysql8" for all patches related to this spec.

.. code-block:: bash

    git-review -t mysql8


Work Items
----------

The implementation consists of the following charms:

- mysql-innodb-cluster

  The primary charm that will handle the mysql implementation.

   - Install MySQL 8
   - Handle InnoDB clustering


- interface-mysql-innodb-cluster

  The peer relationship for mysql-innodb-cluster


- interface-mysql-shared provides

  The interface mysql-shared provides side


- mysql-router

  The subordinate charm that will relate to the application charm and to
  mysql-innodb-cluster.

  - Proxies DB requests and responses


- interface-mysql-router

   The interface between the application and mysql-router


- mysqlsh snap

  The mysqlsh tool that manages the mysql innodb cluster

  - snapped from upstream source


Repositories
------------


- charm-mysql-innodb-cluster

- charm-interface-mysql-innodb-cluster

- charm-mysql-router

- charm-interface-mysql-router

- mysqlsh-snap

Documentation
-------------

Each charm and interface will contain a README with instructions on deploying
and relating the charm.

Potentially an OpenStack deployment guide appendix to discuss clustering and
cluster management.


Security
--------

MySQL 8 will now be in main with security auditing which percona-cluster never
had.

The code reviews of the work items should include a security conscious review
approach.


Testing
-------

Unit tests for each repository.

Zaza functional tests for each charm. This will include end to end testing with
applications, mysql-router and mysql-inndob-cluster.


Dependencies
============

- Eoan+ mysql8 packaging
- MySQL shell
