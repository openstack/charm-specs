..
  Copyright 2017 Canonical LTD

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===================
Keystone Federation
===================

Keystone can be configured to integrate with a number of different identity
providers in a number of different configurations. This spec attempts to
discuss how to implement pluggable backends that utilise Keystone
federations.

Problem Description
===================

When deploying the OpenStack charms to an enterprise customer they are likely
to want to integrate OpenStack authentication with an existing identity
provider like AD, LDAP, Kerberos etc. There are two main ways that Keystone
appears to achieve this integration: backends and federation. Although this
spec is concerned with federation it is useful to go over backends to in an
attempt to distinguish the two.

The following are not covered here:

- Integration with Kerberos
- Horizon SSO
- Federated LDAP via SSSD and mod_lookup_identity
- Keystone acting as an IdP in a federated environment.

Backends
--------

When keystone uses a backend it is Keystone itself which knows how to manage
that backend, how to talk to it and how to deal with operations on it. This
limits the number of backends that keystone can support as each new backend
needs new logic in Keystone itself. This approach also has negative security
implications. Keystone may need an account with the backend (an LDAP username
and password) to perform lookups, these account details will be in clear text
in the keystone.conf. In addition, all users passwords with flow through
keystone.

The keystone project highlights SQL and LDAP (inc AD) as their supported
backends. The status of support for these is as follows:

- SQL: Currently supported by the keystone charm.
- LDAP: Currently supported by the keystone and keystone-ldap subordinate.

These backends are supported with the Keystone v2 API if they are used
exclusively. To support multiple backends then Keystone v3 API needs to be used
and each backend is associated with a particular Keystone domain. This allows
for service users to be in SQL but users to be in ldap for example.

Adding a new backend is achieved by writing a keystone subordinate charm and
relating it to keystone via the keystone-domain-backend interface.

Enabling a backend tends to be achieved via the keystone.conf

Federation
----------

With federation Keystone trusts a remote Identity provider. Keystone
communicates with that provider using a protocol like SAML or OpenID connect.
Keystone relies on a local Apache to manage communication and Apache passes
back to keystone environment variables like REMOTE_USER. Keystone is abstracted
from the implementation details of talking to the identity provider and never
sees the users password. When using Federation, LDAP may still be the ultimate
backend but it is fronted by something providing SAML/OpenID connectivity like
AD federation service or Shibboleth.

Each Identity provider must be associated with a different domain within
keystone. The keystone v3 API is needed to support federation.

Compatible Identity Providers:
(https://docs.openstack.org/ocata/config-reference/identity/federated-identity.html#supporting-keystone-as-a-sp
):

- OpenID
- SAML


Proposed Change
===============

Both Keystone backends and federated backends may need to add config to the
keystone.conf and/or the Apache WSGI vhost. As such, it makes sense for both
types to share the existing interface particularly as the existing interface
is called keystone-domain-backend which does not differentiate between the
two.

This spec covers changes add support for federation using either SAML or
OpenID, to the keystone charm. This will involve extending the
keystone-domain-backend interface to support passing configuration snippets to
the Apache vhosts and creating subordinate charms which implement OpenID and
SAML.

Alternatives
------------

- Add support for federation via OpenID and SAML directly to the keystone
  charm.
- Create a new interface for federation via OpenID and SAML

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  None


Gerrit Topic
------------

Use Gerrit topic "keystone_federation" for all patches related to this spec.

.. code-block:: bash

    git-review -t <keystone_federation>

Work Items
----------

**Create deployment scripts to create test env for OpenID or SAML integration**

**Extend keystone-domain-backend**

The keystone-domain-backend interface will need to provide the following:

- Modules for Apache to enable
- Configuration for principle to insert into Apache keystone wsgi vhosts
- Subordinate triggered restart of Apache
- Auth method(s) to be added to keystone's [auth] methods list
- Configuration for principle to insert into keystone.conf

**Configure Keystone to consume new interface**

Keystone charm will need to be updated to respond to events outlined in the
interface description above

**New keystone-openid and keystone-saml subordinates**

The new subordinates will need to expose all the configuration options needed
for connecting to the identity provider. It will then need to use the
interface to pass any required config for Apache or Keystone up to the
keystone principle.

Repositories
------------

New projects for the interface and new subordinates will be needed.

Documentation
-------------

This will require documentation in the READMEs of both the subordinates and
the keystone charm. A blog walking through the deployment and integration
would be very useful.

Security
--------

Although a Keystone back-end will determine who has access to the entire
OpenStack deployment, this specific charm will only change Keystone and Apache
parameters, avoiding default values and leave the configuration to the user
should be enough.

Testing
-------

The code must be covered by unit tests. Ideally amulet tests would be extended
to cover this new functionality but deploying a functional openid server for
keystone to use may not be practical. It must be covered by a Mojo spec though.

Dependencies
============

None
