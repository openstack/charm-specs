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

=============================================
Enablement of Keystone OpenID Connect Support
=============================================

Keystone Charm support pluggable backends that use Keystone federations to
enable a variety of providers for the account users and groups. This spec
attempts to discuss how to enable support for `OpenID Connect`_.

Problem Description
===================

When deploying the OpenStack charms in an enterprise environment currently
it's only possible to bring up the central database of users into Keystone via
LDAP or Federation, this last one method support only SAML integration, hence
environments where users are exposed with OpenID Connect (e.g. Google Apps),
Charmed OpenStack provides no solution.

Proposed Change
===============

The proposed solution is to make OpenID Connect available to Charmed OpenStack
in a new subordinate charm to keystone.

The new feature would be supported on OpenStack Yoga (and later) on Ubuntu
22.04 LTS.

Alternatives
------------

- Allow operators to configure OpenID Connect federation with some support
  backed into the Keystone charm directly.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Felipe Reyes <felipe.reyes@canonical.com>


Gerrit Topic
------------

Use Gerrit topic "keystone-openidc" for all patches related to this spec.

.. code-block:: bash

    git-review -t keystone-openidc

Work Items
----------

* Implement a new operator framework subordinate charm, keystone-openidc, that
  implements keystone-fid-service-provider and websso-fid-service-provider
  relations

* Add a charm option that allows the operator to pass a custom HTML template
  to be used as a Single Sign On template, this template will be used by
  OpenStack Horizon to render the login screen (see `Configuring Horizon as a
  WebSSO Frontend`_)

* Create a new snap with a OpenID Connect provider software such as Ipsilon_
  or Oidc-op_, this snap will be designed with the intention of being deployed
  and used for testing purposes.

* Create a new charm that deploys a OpenID Connect provider for testing
  purposes, openidc-test-fixture, this charm will be similar to
  ldap-test-fixture_.

* Write unit tests

* Extend current functional tests using the zaza testing framework

Timeline
--------

The goal is to implement this change in the OpenStack Charms 22.04 release.

Repositories
------------

During the intial development the code will be managed here:

- https://github.com/openstack-charmers/charm-keystone-openidc
- https://github.com/openstack-charmers/charm-openidc-test-fixture
- https://github.com/openstack-charmers/snap-<openidc-provider>

When ready the charm keystone-openidc will be managed via OpenDev gerrit at:

https://opendev.org/openstack/charm-keystone-openidc

Documentation
-------------

The charm should contain documented options:

* Create charm options

* Create charm relations

* Create charm README

* Update the deployment-guide_ to include a section that explains how to
  configure Charmed OpenStack to authenticate against a OpenID Connect
  provider.

Security
--------

The OpenStack Identity Service (Keystone) has to interact exclusively with
OpenID Connect providers served over valid HTTPS endpoints.

Testing
-------

Code changes will be tested by unit tests.

Functional tests would be covered by zaza testing framework.

Dependencies
============

- This enablement depends on libapache2-mod-auth-openidc_ available in Ubuntu
  since 18.04 (Bionic) in the Universe pocket. To maintain the security and
  availability of this component in the long term a Main Inclusion Request
  (MIR) will be submitted.

.. _OpenID Connect: https://openid.net/connect/
.. _Ipsilon: https://pagure.io/ipsilon
.. _Oidc-op: https://github.com/IdentityPython/oidc-op
.. _deployment-guide: https://opendev.org/openstack/charm-deployment-guide/
.. _libapache2-mod-auth-openidc: https://launchpad.net/ubuntu/+source/libapache2-mod-auth-openidc
.. _Configuring Horizon as a WebSSO Frontend: https://docs.openstack.org/keystone/latest/admin/federation/configure_federation.html#add-the-callback-template-websso
.. _ldap-test-fixture: https://github.com/openstack-charmers/charm-ldap-test-fixture
