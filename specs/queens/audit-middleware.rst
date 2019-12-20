..
  Copyright 2019 Canonical UK Ltd

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

This is a requirement from one of the customers to enable audit middleware

Problem Description
===================

Currently we have to make manual changes to the configuration to enable audit
middleware, this will enable us to automate this process across all the
OpenStack projects

Proposed Change
===============

Update existing charms to enable this feature

The customer in question is currently running bionic queens, this spec is a
basis for that request

Alternatives
------------

Do it manually, which is not feasible

Implementation
==============

1. Understand the changes required for each project, maybe by changing by hand
1. If there are common changes, then charmhelpers may be used to make these
   changes
1. Write tests in charmhelpers for these changes
1. For each of the projects
  1. sync the new charmhelpers
  1. Add the relevant updated templates
      * `/etc/<project>/<project>.conf`
      * `/etc/<project>/api-paste.ini`
      * `/etc/<project>/api_audit_map.conf`
  1. write the amulet or zaza tests to ensure that the changes are good

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

Same as #implementation above

Repositories
------------

No new git repositories will need to be created. However, multiple git
repositories will need to be touched for this implementation to work

It will depend on the following repo, which will have the `api_audit_map.conf`
file

https://github.com/openstack/pycadf/tree/master/etc/pycadf

At first glance the following repositories will be affected that are part of the
OpenStack projects, this is based on the files that are listed in the above
repository

* https://github.com/openstack/charm-nova-cloud-controller
* https://github.com/openstack/charm-glance
* https://github.com/openstack/charm-cinder
* https://github.com/openstack/charm-gnocchi
* https://github.com/openstack/charm-heat
* https://github.com/openstack/charm-ironic
* https://github.com/openstack/charm-neutron-api
* https://github.com/openstack/charm-panko

The following repo may also need to be updated, so ensure that similar
information is stored in one central plae, rather than duplicating the contents
in the above repositories

* https://github.com/juju/charm-helpers

Initial work was tried in the following commits

* https://github.com/arif-ali/charm-nova-cloud-controller/commit/3743f00384de56efe8b0a4ee2ab2e40de68b5e7f
* https://github.com/arif-ali/charm-helpers/commit/258cf87c83cca2faf601dd99285cd226e2e67b48


Documentation
-------------

Will this require a documentation change?  If so, which documents?
Will it impact developer workflow?  Will additional communication need
to be made?

Identify the surface area of doc updates explicitly, ie. charm-guide,
deployment-guide, charm README, wiki page links.

Security
--------

Unknown

Testing
-------

charmhelpers: tests will be needed to ensure that the extra options that are
added will work

All the projects, functional amulet/zaza tests will need to be added for the
new option, and ensuring the the configuration is changed correctly, and the
audit messaging is getting through to the message bus and/or being logged

Dependencies
============

Other than the ones specified above, further dependencies are unknown
