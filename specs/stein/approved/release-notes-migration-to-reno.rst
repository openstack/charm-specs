..
  Copyright 2018 Canonical Ltd.

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
Release Notes Migration to Reno
===============================

The OpenStack Charms project has a ever increasing velocity with improvements
and new features being committed by disperse group of people each cycle.

Currently we manage our release notes in a central document and it is typically
put together manually at the end of each cycle.

Problem Description
===================

Putting together the release notes document involves manual labor and the
current process is error prone with high risk of leaving important information
out of the release notes at release time.

The upstream OpenStack release process has been heavily automated in recent
cycles and we are currently barred from announcing our release notes to the new
release-announce mailing list.  Our repositories must be aligned with upstream
automated release tools to gain access.  Modernizing the way we do release
notes is one of the steps needed to accomplish this.

Proposed Change
===============

To address these issues I propose we do a stepwise migration towards committing
release notes along with code changes throughout our charm repositories.

Prior work we can make use of exists upstream in the Reno [0] project, with
developer tooling, linters and gate jobs. The use of this methodology and
tooling is widespread among other OpenStack projects.

As a first step I propose we enable the use of reno on the following
repositories during the cycle leading up to the 18.11 Charm release:

- charm-keystone
- charm-nova-compute

After the 18.11 Charm release we evaluate and decide on further adoption.

0: https://docs.openstack.org/reno/latest/

Alternatives
------------

None defined.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fnordahl

Gerrit Topic
------------

Use Gerrit topic "charm-reno" for all patches related to this spec.

.. code-block:: bash

    git-review -t charm-reno

Work Items
----------

- Add necessary boiler plate for use with reno to affected charm repositories

- Add tox environment that enables developer to run reno tooling from virtual
  environment, relieving developer from the responsibility of installing and
  maintaining it in their developer environment.

- Add tox environment that enables gating on existence and correctness of
  release notes artifacts.

- Research ways to best coordinate and extract release notes artifacts from
  multiple charm repositories and combine them in one (charm-guide) at release
  time.

- Create preliminary guidelines for use of developers and reviewers.

Repositories
------------

No new git repositories are needed.

Documentation
-------------

Preliminary guidelines for use of developers and reviewers must be created.
Communication with the community is also necessary.

Security
--------

This change does not introduce any additional security risks.

Testing
-------

The process for creating release notes files and committing them with code
changes will be discussed with the community as it is implemented.

Gate jobs checking for existence and correctness of the release notes will be
implemented.

Dependencies
============

Reno - https://docs.openstack.org/reno/latest/
