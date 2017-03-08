..
  Copyright 2016, Canonical UK

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===========================
Keystone Federation Support
===========================

Keystone Federation is a maturing feature and charm support for it is
frequently requested.

Problem Description
===================

Single identity across a cloud with multiple, geographically disparate
regions is complex for operators; Keystone federation provides the
ability for multiple clouds to federate identity trusty between
regions, supporting single identity either via keystone or via a 3rd
party identity provider. The use of Keystone Federation will help us
build bigger, more manageable clouds across geographies.


Proposed Change
===============

The design of this solution is predicated on the use of the Keystone
federation features introduced in OpenStack Kilo; these allow Keystone
to delegate authentication of users to a different identity provider
(IDP) which might be keystone, but could also be a solution implementing
one of the methods used for expressing assertions of identity (saml2 or
OpenID).

If IDP is keystone, then this is the current straw man
design:

- A single ‘global’ keystone is set-up as the IDP for the cloud; this
  service provides authentication for users and the global service
  catalog for all cloud regions.
- Region level keystone instances delegate authentication to the global
  keystone, but also maintain a region level service catalog of endpoints
  for local use

An end-user accesses the cloud via the entry point of the global keystone;
at this point the end-user will be redirected to the region level services
based on which region they which to manage resources within.

In terms of charm design, the existing registration approach for services
in keystone is still maintained, but each keystone deployment will also
register its service catalog entries into the global keystone catalog.

They keystone charm will need updating to enable a) operation under apache
(standalone to be removed in mitaka and b) enablement of required federation
components.  There will also be impact onto the openstack-dashboard charm to
enable use of this feature.

There is also a wider charm impact in that we need to re-base onto the
keystone v3 api across the board to support this type of feature.

The packages to support identity federation will also need to be selected
and undergo MIR into Ubuntu main this cycle; various options exist:

- SAM: Keystone supports the following implementations:
       Shibboleth - see Setup Shibboleth.
       Mellon - see Setup Mellon.
- OpenID Connect: see Setup OpenID Connect.

The Keystone Federation feature should support:

- Federation between two keystone services in the same model
- Federation between two keystone services in different models
- Federation between keystone and an identity provider not managed by
  Juju

Alternatives
------------

Identities can be kept in sync between keystone instances using database
replication. The keystone charm also supports using LDAP as a backend,
keystone charms in different models could share the same LDAP backend if
there service users are stored locally.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gnuoy

Gerrit Topic
------------

Use Gerrit topic "keystone-federation" for all patches related to this
spec:

.. code-block:: bash

    git-review -t keystone_federation

Work Items
----------

Keystone Investigative Work
+++++++++++++++++++++++++++

- Deploy multiple juju environments and define one IDP keystone and the
  rest as SPs. Configuration will be manually applied to units
- Test OpenID integration with keystone. Configuration will be manually
  applied to units

Keystone v3 endpoint enablement
+++++++++++++++++++++++++++++++

- Define intercharm protocol for agreeing keystone api version
- Enable Keystone v3 in keystone charm
- Enable Keystone v3 in client charms
- Update Openstack charm testing configuration scripts  to talk keystone
  v3
- Create Mojo spec for v3 deploy


Keystone to keystone Federation enablement
++++++++++++++++++++++++++++++++++++++++++

- Switch keystone to use apache for all use cases on deployments >= Liberty
- Enable keystone to keystone SP/IDP relation using a config option in the
  charm to define the IDP endpoint (in lieu of cross environment relations)
- Mojo spec to deploy two regions and test federated access

Keystone to OpenID 3rd party enablement
+++++++++++++++++++++++++++++++++++++++

- Backport libapache2-mod-auth-openidc to trusty cloud archive
- Expose OpenID configuration options to keystone charm, and update keystone
  apache accordingly.
- Create bundle for deploying a single region using UbuntuONE for
  authentication.
- Mojo spec for multi-region UbuntuONE backed deployment

Keystone to SAML 3rd party enablement
+++++++++++++++++++++++++++++++++++++

- Expose SAML configuration options to keystone charm, and update keystone
  apache accordingly.
- Create bundle for deploying a single region using SAML for authentication.
- Mojo spec for multi-region SAML backed deployment

Repositories
------------

No new repositories

Documentation
-------------

The Keystone charm README will be updated with instructions for enabling
federatioon. A blog post is optional but would be a useful addition.

Security
--------

Security review may be required.

Testing
-------

Code changes will be covered by unit tests; functional testing will be done
using a combination of Amulet, Bundle tester and Mojo specification.

Dependencies
============

None
