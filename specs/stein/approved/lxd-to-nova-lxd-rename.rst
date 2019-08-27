..
  Copyright 2019 Canonical Limited

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

==================================
Renaming the lxd charm to nova-lxd
==================================

The initial 'push' to rename the charm to *nova-lxd* is:

1. The name *nova-lxd* better reflects that the charm is used with nova.
   It is confusing for users that the (currently named) *lxd* charm must
   be used with the *nova-compute* charm, and can't be used in a different
   environment.
2. It releases the name *lxd* for use as a general purpose charm for LXD,
   that could be created to allow clusters of LXD machines to configured
   with Juju and MaaS.

Problem Description
===================

The name of the charm as ``lxd`` is causing some confusion in the
community as potential users are unaware that it is an OpenStack specific
charm for nova as a subordinate.  Therefore, changing the name of the
charm to ``nova-lxd`` will help mitigate this confusion.

Proposed Change
===============

- The ``lxd`` charm will be copied to ``nova-lxd`` with all of its existing
  functionality.
- The existing ``lxd`` charm will be converted to a no-op charm with a blocked
  status message indicating that the user should use the ``nova-lxd`` charm
  instead.  The ``lxd`` charm will then be removed entirely after another
  cycle.
- Making the existing ``lxd`` charm a no-op also has the advantage that
  existing installations will be *unable* to upgrade to the newer 'no-op' lxd
  charm as the relations will not match, thus forcing the operator to
  investigate as well as not breaking existing installations.

The charm name will be switched with the ``juju upgrade-charm lxd
--switch=nova-lxd`` method of changing a charm and this will be tested
during upgrades.

Alternatives
------------

No alternatives suggested.

Implementation
==============

Assignee(s)
-----------

Primary assignee: ajkavanagh

Gerrit Topic
------------

Use Gerrit topic "nova-lxd-charm" for all patches related to this spec.

.. code-block:: bash

    git-review -t nova-lxd-charm

Work Items
----------

There are the following, logically separate, pieces of work in this
specification:

1. Turning the existing ``lxd`` charm into a 'no-op' charm with a *blocked*
   status message.  For the avoidance of doubt, Juju will *refuse* to upgrade
   an existing ``lxd`` charm to this charm.  The purpose of the *blocked*
   message is for new deployments where the deployment bundle *should* use
   ``nova-lxd``, but instead refers to the old name.  In this case, the charm
   will not do anything as a subordinate except to show the *blocked* message.
2. Creating the new ``nova-lxd`` charm by copying the existing ``lxd``
   charm repository over.
3. Create project for nova-lxd charm on OpenStack on gerrit
4. Create launchpad project for nova-lxd charm

Repositories
------------

New git repository:  ``openstack/charm-nova-lxd``

This will be created as part of creating the charm-nova-lxd project on
openstack.

Documentation
-------------

1. Add notes the Charm Deployment Guide about the new charm.
2. Add notes to the upgrades section to discuss how to upgrade from
   ``lxd`` to ``nova-lxd``, what issues may occur, and how to recover the
   situation.

Security
--------

None noted.

Testing
-------

As the charm is just being renamed, there should be no issues with unit and
functional tests.  They will not change as part of the specification.

However, the various bundles that use the ``lxd`` charm will need to be updated
to refer to a ``nova-lxd`` charm for the 19.04 release.  These are contained
in:

- github: openstack-charmers/openstack-bundles - openstack-lxd
- github: openstack-charmers/openstack-charm-testing
  - README-lxd.md
  - README-multihypervisor.md
  - bundles/lxd/*
  - bundles/multi-hypervisor/*

Testing on a MaaS will be necessary for the modified openstack-lxd bundles.

Dependencies
============

None noted.
