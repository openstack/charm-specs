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

=============
Octavia Charm
=============

At present our support for LBaaS makes use of the ``neutron-lbaas``
*service_plugin* and *service_provider* drivers.  In the existing
implementation the load balancer code, which is a part of the datapath, is run
in namespaces directly on the ``neutron`` gateway units.

The ``neutron-lbaas`` and ``neutron-lbaas-dashboard`` projects are marked as
deprecated as of the Queens OpenStack release cycle and has stopped receiving
updates.  They will be removed entirely at some point in the future.

Our usage of ``neutron-lbaas`` driver also gives a sub-optimal configuration
for users in need of TLS.

Proposed Change
===============

The ``neutron-lbaas`` projects have been replaced by ``octavia``,
``octavia-dashboard``, and ``python-octaviaclient``.

``octavia`` has a separate API endpoint and implements a superset of the
LBaaS v2 API.  For migration of existing users there is a ``lbaasv2-proxy``
service plugin which allows ``neutron`` API endpoint to forward LBaaS v2 API
calls to the ``octavia`` endpoint in a migration period.

Contrary to the ``neutron-lbaas`` implementation, ``octavia`` runs the
in-datapath load balancer code as instances in your cloud.  This gives richer
flexibility both in terms of freedom of choice of provider of the load balancer
software itself and dynamic scaling of the service.

On the scaling side, most if not all, of the ``octavia`` services benefit from
being scaled out proportional to the number of running load balancers and load
balancer life cycle events.  Thus it makse sense to co-locate the API, Worker,
Health Manager and Housekeeping Manager daemons in the same charm unit, and
scale by increasing the number of charm units deployed.

A reference implementation load balancer based on ``Ubuntu`` cloud images
running ``HAProxy`` called ``amphorae`` is available.

Alternatives
------------

As our current dependencies for providing LBaaS are deprecated and set to be
removed at a future date there are no real alternatives to doing this work.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  fnordahl

Gerrit Topic
------------

Use Gerrit topic "octavia-charm" for all patches related to this spec.

.. code-block:: bash

    git-review -t octavia-charm

Work Items
----------

- Implement a new principal ``octavia`` charm which will deploy the ``octavia``
  API, Worker, Health Manager and Housekeeping Manager daemons.  The charm MUST
  have all the characteristics you expect from a OpenStack API charm such as
  TLS, HA, pause / resume actions, upgrades etc.

- Make necessary changes to ``neutron`` charms for operation with ``octavia``.

- Make necessary changes to ``openstack-dashboard`` charm for replacing the
  ``neutron-lbaas-dashboard`` with the ``octavia-dashboard``.

- Implement support for storing TLS private key secrets in ``Vault`` either
  through relation to ``barbican`` or by utilizing the ``castellan`` library
  directly.

- Implement support for doing post-deployment configuration and plumbing of the
  private administration network between ``octavia`` and the load balancer
  instances it deploys.  This will be done by interacting with the ``neutron``
  API and deploying the ``neutron-openvswitch`` subordinate with the charm.
  The ``neutron-openvswitch`` subordinate will automatically take care of
  creating necessary tunnels for access to the overlay network and we can
  subsequently create ``OpenvSwitch`` ports on the units where ``octavia``
  services are deployed.

- Provide documentation and/or any actions necessary for obtaining,
  provisioning or indeed building (if necessary) the ``amphorae`` load balancer
  software images.

- Provide documentation and/or any actions necessary for migrating existing
  load balancer workloads to ``octavia``.

Repositories
------------

A new git repository will be required for the ``octavia`` charm:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-octavia

Documentation
-------------

The ``octavia`` charm should contain a README with instructions on deploying
the charm.

Security
--------

- Serving TLS terminated traffic is a basic feature of a load balancer service.
  With that comes responsibility for secure handling and storage of sensitive
  information such as private keys.

  Best practices for handling and storing these keys will be implemented.

- If we are to provide our user with a mechanism to build the ``amphorae``
  images, it must be made clear that the responsibility of keeping the built
  image secure and up to date rests solemnly with our user.

- The presence of a direct network tunnel between the ``octavia`` unit placed
  in the undercloud control plane and the load balancer instances in the
  overcloud is a construct we have not encountered previously.  We must
  consider all aspects of this carefully with security and integrity in mind.

Testing
-------

Code written or changed will be covered by unit tests; functional testing will
be done using ``Zaza``.

Dependencies
============

- To be able to support deployment of ``octavia`` charm in ``LXD`` containers
  we depend on ``Juju`` implementation of charm controlled ``LXD`` profile
  updates specification_.

  .. _specification: https://discourse.jujucharms.com/t/wip-specification-for-lxd-profile-updates-permitted-by-charms/78
