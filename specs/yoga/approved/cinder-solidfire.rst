..
  Copyright 2022 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

=========================================
Cinder subordinate charm cinder-solidfire
=========================================

Problem Description
===================

Cinder has a great number of drivers for different backends. Almost all
currently available commercial storage arrays and many other software only
solutions have a driver available, if not already in the upstream cinder
project, as an installable package that enables cinder with the extra driver.

Because of how many possibilities exist, a model was developed where a separate
subordinate charm will be deployed to take care of the specifics of the driver
and pass configuration to the main cinder charm over a relation so that cinder
can write to its own configuration file.

Proposed Change
===============

Instead of changing the cinder charm, this proposal aims to create a separate
subordinate charm specific for the solidfire driver. This cinder
installation would enable access to Solidfire devices. The following points
are of condiderable importance for the development of this subordinate charm:

* The new implementation should not conflict with current implementation. Both
  should co-exist, even in the same deployment if possible.
* The new charm should implement all features currently existing in the cinder
  charm, regarding the solidfire functionality.

Note that because the Solidfire driver is part of the cinder-common package, no
additional packages need to be installed for this charm to work.

This charm would be supported in the Ussuri release of Openstack and newer.

Alternatives
------------

At the time of this writing, there exist no alternatives.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
    Gustavo Sanchez <gustavo.sanchez@canonical.com> - LP ID: gustavosr98

Secondary assignee:
    Luciano Lo Giudice <luciano.logiudice@canonical.com> - LP ID: lmlogiudice

Gerrit Topic
------------

Use Gerrit topic "charm-cinder-solidfire" for all patches related to this spec.

.. code-block:: bash

    git-review -t charm-cinder-solidfire

Work Items
----------

The charm, as listed above. Plus, both unit and functional tests will be
written.

Repositories
------------

Presently, the following repository hosts the charm code:

https://opendev.org/openstack/charm-cinder-solidfire

Documentation
-------------

Once this charm is completed, charm-guide will be updated to reference it.

Security
--------

No security aspect is going to change in relation to the current solution.

Testing
-------

Code changes will be covered by both unit and functional tests.
Functional tests would require Solidfire storage hardware.

Dependencies
============

No dependencies in addition to the ones that exist in the current solution.
