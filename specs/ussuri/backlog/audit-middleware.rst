..
  Copyright 2019 Canonical Ltd

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

==========================================================
Enable Audit Middleware that comes with keystonemiddleware
==========================================================

This is a requirement from one of the customers to enable audit middleware.

Problem Description
===================

Currently, manual changes are made to the configuration to enable audit
middleware.  This specification is for a configuration option that can be used
to enable audit middleware in a charm.  This can be applied, as required, to
applicable OpenStack charms.

Proposed Change
===============

Update existing charms to enable this feature.

The customer in question is currently running bionic queens. This spec is a
basis for that request.

Alternatives
------------

Do it manually.

Implementation
==============

For each of the OpenStack charms that provides API, we need to do the
following:

* Add a configuration option to enable or disable audit middleware.
* We need to add the specific sections that need to go into 3 files.

   - ``/etc/<project>/<project>.conf``
   - ``/etc/<project>/api-paste.ini``
   - ``/etc/<project>/api_audit_map.conf``

* Test to see if the corresponding files are changed correctly.
* Write unit and functional tests.

Templates for ``/etc/<project>/api_audit_map.conf`` file can be found in
https://github.com/openstack/pycadf/tree/master/etc/pycadf.

For further details on the implementation see
https://docs.openstack.org/keystonemiddleware/latest/audit.html.

Assignee(s)
-----------

Primary assignee:
  None

Gerrit Topic
------------

Use Gerrit topic "audit-middleware" for all patches related to this spec.

.. code-block:: bash

    git-review -t audit-middleware

Work Items
----------

#. Understand the changes required for each project, maybe by changing by hand.

#. Common changes will be implemented in the charmhelpers library.

#. Write tests in charmhelpers for these changes.

#. For each of the projects:

   #. sync the new charmhelpers.

   #. Add the relevant updated templates.

      - ``/etc/<project>/<project>.conf``

      - ``/etc/<project>/api-paste.ini``

      - ``/etc/<project>/api_audit_map.conf``

   #. Write the amulet or zaza tests to ensure that the changes are good.

Repositories
------------

No new git repositories will need to be created. However, multiple git
repositories will need to be touched for this implementation to work

These are the initial charms that are within the scope of this specification:

* https://github.com/openstack/charm-nova-cloud-controller
* https://github.com/openstack/charm-glance
* https://github.com/openstack/charm-cinder
* https://github.com/openstack/charm-gnocchi
* https://github.com/openstack/charm-heat
* https://github.com/openstack/charm-ironic
* https://github.com/openstack/charm-neutron-api
* https://github.com/openstack/charm-panko

The following repo will also need to be updated, so ensure that similar
information is stored in one central place, rather than duplicating the
contents in the above repositories.

* https://github.com/juju/charm-helpers

Initial work was tried in the following commits:

* https://github.com/arif-ali/charm-nova-cloud-controller/commit/3743f00384de56efe8b0a4ee2ab2e40de68b5e7f
* https://github.com/arif-ali/charm-helpers/commit/258cf87c83cca2faf601dd99285cd226e2e67b48

Documentation
-------------

It will be documented within each of the charms' ``config.yaml``.

Security
--------

Enable API auditing for security compliance.

Testing
-------

* Unit tests will be added to charm-helpers.
* Functional tests will need to be added for the new option, and checking that
  the configuration is changed correctly, and then disabled.

Dependencies
============

There are no further dependencies.
