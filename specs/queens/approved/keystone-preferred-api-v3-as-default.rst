..
  Copyright 2017 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===============================================
Keystone - Change default preferred API version
===============================================

The current default preferred API version used by the Keystone Charm is v2.0.

There exists multiple use cases where the use of the v3 API is required.
To use features such as domains, domain specific drivers, LDAP authentication,
Federation (OpenID Connect, SAML) and others not mentioned, the Keystone V3
API endpoints must be exposed in the catalog and all services in a deployment
must leverage the V3 API when authenticating with Keystone.  Increasingly so
these use cases would be the desired mode of operation for new OpenStack
deployments.

Additionally the v2.0 API has been marked as DEPRECATED since the OpenStack
Mitaka release and will be REMOVED in the Queens release cycle.

Problem Description
===================

The Keystone charm currently has two separate code paths for managing the
Keystone service.  Which path is exercised depends on the value of the
'preferred-api-version' setting.  This leads to duplication of both code, and
potential bugs.

The majority of our functional tests are currently run with services configured
to talk to Keystone using the v2.0 API.

Change of endpoints in the Keystone catalog and reconfiguration of services to
use the v3 API automatically in already running deployments may lead to
unforeseen and undesirable side effects.  We must find a way to have the change
of default affect new deployments only, and leave existing deployments
untouched on charm upgrade.

Proposed Change
===============

Remove API v2.0 specific code from 'hooks/manager.py' and make use of a
combination of the 'keystone-manage' CLI tool and the v3 API to manage Keystone
service regardless of the value of 'preferred-api-version' configuration
option.

Re-wire existing unit- and functional- tests to be performed on deployments
configured to use both v2.0 and v3 API.  This induces change in both the
keystone charm itself as well as any other charm with specific tests for how
it relates to the keystone charm.

Remove default value for 'preferred-api-version' in config.yaml and let charm
decide which default to use programatically.  The upgrade function should be
made to retain 'preferred-api-version' as 2 for existing deployments that makes
use of the current default.

Version fence will be implemented in charm to have the new default apply for
OpenStack versions 'Queens' and onwward.

Alternatives
------------

There are no real alternatives.  Not changing the default would make our test
coverage poorer for the real world use cases that our software is currently
exposed to.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fnordahl

Gerrit Topic
------------

Use Gerrit topic "keystone-v3" for all patches related to this spec.

.. code-block:: bash

    git-review -t keystone-v3

Work Items
----------

- Update 'hooks/manager.py' to use a combination of keystone-manage CLI and
  v3 API to manage the Keystone deployment.  Remove any v2.0 API specifc code.
  This work item should include removal of dependency on having admin-token in
  Keystone configuration file and to switch charm over to use keystoneauth1
  with sessions.
- Update relevant functions in charmhelpers.contrib.openstack.amulet.utils
- Make all existing unit- and functional- tests run on both v2.0 API and v3 API
  configured deployments in the gate.
- Implement version fence that applies the new default for OpenStack release
  'Queens' and onward.
- Remove default value for 'preferred-api-version', make decision in charm
  programatically.
- Make sure simplestreams with Keystone v3 API support is backported to our
  stable releases.

Repositories
------------

No new git repositories required.

Documentation
-------------

This change calls for an update of the documentation in README.md with
information about how the charm sets up domains and projects in a v3 API
enabled Keystone deployment. The documentation should also include examples of
openrc files and simple openstack client usage.

Security
--------

We are exercising existing code paths and are changing the default value of
how the Keystone deployment is configured to match real world usage, we are not
introducing new changes that have security implications.  Revisiting and
re-validating the security of the existing implementation is in place as a part
of this work.

Testing
-------

Implementation will update existing unit- and functional- tests to leverage
both v2.0 and v3 API configurations.  Scripts, bundles and specs used in
periodic and pre-release testing should also be updated to handle and
excercise the new default.

Dependencies
============

No external dependencies.
