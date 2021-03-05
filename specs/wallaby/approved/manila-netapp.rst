..
  Copyright 2021 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===========================================
NetApp Data ONTAP as Manila storage backend
===========================================

The existing Manila NetApp Clustered Data ONTAP driver should be used to
configure the shares storage backend.

Problem Description
===================

There isn't any backend configuration charm that relates to the principal
Manila charm, and configures the NetApp storage backend.

Proposed Change
===============

A new subordinate charm that relates to the Manila principal charm, and sends
the proper NetApp Data ONTAP backend configuration.

The new charm is a backend configuration charm that allows relevant config
options to connect to an already deployed NetApp Data ONTAP cluster. The config
options need to suffice for the Manila backend configuration.

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ionutbalutoiu

Gerrit Topic
------------

Use Gerrit topic "manila-netapp" for all the patches related to this spec.

.. code-block:: bash

    git-review -t manila-netapp

Work Items
----------

- manila-netapp

  - Brand new subordinate charm that implements the ``manila-plugin``
    interface, and it relates to the principal Manila charm. The charm is
    responsible to do the appropriate backend configuration to have
    NetApp Data ONTAP as Manila shares storage.

Repositories
------------

- openstack/charm-manila-netapp

Documentation
-------------

This change will require new documentation to the charm-guide, in addition
to new charm documentation.

Security
--------

The Manila NetApp  driver needs the admin credentials to manage the external
storage cluster. These credentials will be stored in the charm config, and
they will be part of the Manila backend configuration.

Testing
-------

Unit tests will be added to cover the new charm functionality.

New functional tests will be added to validate the new charm, including
end-to-end testing of the entire solution.

Dependencies
============

The packages that are required for this work are already included in the
Ubuntu archives. There are no other new, external dependencies.
