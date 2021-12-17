..
  Copyright 2020, Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

======
Ironic
======

Ironic is the OpenStack bare metal provisioning project, which aims to
provision bare metal servers instead of virtual machines. It integrates
well with existing OpenStack deployments and can function along side
existing Nova deployments, enabling both virtual and physical deployments
in the same tenancy.


Problem Description
===================

There is currently no charm that lets you easily deploy Ironic as part of
Charmed OpenStack.

Proposed Change
===============

Implement a set of reactive charms that enables the deployment of all required
Ironic components. Changes will also be made to existing charms to integrate
with the new Ironic charms.

Baseline supported OpenStack version will be Train.

Alternatives
------------

None

Implementation
==============

The implementation consists of the following new charms:

- ironic-api

  - Provides the Ironic API. OpenStack administrators and nova-compute will
    call into this API to add, provision, clean and rescue bare metal nodes.

  - Requires relations to AMQP, database and keystone.

- ironic-conductor

  - Manages the lifecycle of bare metal nodes. Implements the various power
    drivers for the bare metal nodes.

  - Provides TFTP server for legacy PXE deployments.

  - Provides a HTTP server for iPXE deployments.

  - Can deploy bare metal nodes using iSCSI deploy (requires conductor to be
    deployed on bare metal) or local deploy (requires Glance with Swift storage
    backend, and temporary URL support).

  - Requires relations to AMQP, database and keystone.

- neutron-api-plugin-ironic

  - Enables tight integration between Ironic and Neutron.

  - Enables the bare metal mechanism driver in neutron-server.

  - Starts a neutron ironic agent.

  - This will be a subordinate charm, attached to the neutron-api charm,
    which will enable the above mentioned features.

The implementation also requires changes to the following charms:


- ceph-radosgw

  - The ``object-store`` interface needs to be added to ``ceph-radosgw`` in
    order to allow ``glance`` to consume object storage as a backend for
    images.

- glance

  - Multi backend support needs to be added to glance. Ironic requires access
    to OS images, stored inside an object storage system that allows the
    creation of `temporary URLs <https://docs.openstack.org/swift/latest/api/
    temporary_url_middleware.html>`_

  - Creating a relation to ``ceph-radosgw`` or ``swift`` will enable ``glance``
    to store images in object storage. Local storage will also be enabled.

  - Order of preference for default backend to store images will be:

    - RBD

    - Object storage

    - Local

  - If a relation to RBD exists, this will be set as a default. If a relation
    to ``object-store`` exists, but not ``RBD``, then swift will be used.
    If no relation exists to either ``RBD`` or ``object-store``, local storage
    will be used.

  - Operators will still be able to explicitly upload to one of the enabled
    storage backends by passing the ``--store`` flag when creating an image.

- nova-compute

  - When not deploying Ironic as a stand-alone service, Nova acts as a
    front-end, and scheduler. A key piece of this enablement is the
    ``ironic virt driver``, which is acts as a client that calls into
    ``ironic-api`` when an operator requests a new bare metal instance.

  - ``nova-compute`` charm will be extended to allow setting the ``ironic``
    virt type. This will enable the proper virt driver in ``nova-compute.conf``
    and add any needed config sections to nova.

  - A relation to ``ironic-api`` will be mandatory when the ironic virt driver
    will be in use. This will block the ``nova-compute`` charm, until the
    ``ironic-api`` will be ready. The nova-compute service will otherwise fail,
    and will need manual intervention to restart it once ``ironic-api`` is
    available.


Assignee(s)
-----------

Primary assignee:
  gabriel-samfira

Gerrit Topic
------------

Use Gerrit topic "ironic-charm" for all patches related to this spec.

.. code-block:: bash

    git-review -t ironic-charm

Work Items
----------

In addition to the items in the Implementation_ section, an ``ironic-api``
interface will also be created.

Repositories
------------

- charm-ironic-api

- charm-ironic-conductor

- charm-neutron-api-plugin-ironic

- charm-interface-ironic-api

Documentation
-------------

Each charm will contain a README with instructions on deploying the charm.

In addition, an appendix will be added to the deployment guide which will
outline various deployment scenarios, hardware and networking requirements
and config options.

Security
--------

- ``ironic-api``

  - Requires keystone, database and rabbitmq credentials which will be
    stored in ``ironic.conf``

  - IPMI/BMC credentials for each bare metal node will be stored in the
    database.

  - The API endpoints for Ironic will optionally be accessible via TLS,
    either by creating a relation to Vault / easy-rsa, or by manually
    supplying the TLS key/cert via config

- ``ironic-conductor``

  - Requires keystone, database and rabbitmq credentials which will be
    stored in ``ironic.conf``

Testing
-------

Code written or changed will be covered by unit tests; functional testing will
be implemented using the ``Zaza`` framework.

Dependencies
============

- A new nova-compute-ironic package will need to be added. Alternatively a
  different virtual driver package can be used, and the appropriate config
  can be create from a template.

  - The nova-compute-ironic will provide an appropriate ``nova-compute.conf``
    with the ironic virtual driver enabled. The package will need to have
    python3-ironicclient as a dependency, in addition to the dependencies
    that come from python3-nova.

- The networking-baremetal packages will need to be made available for all
  supported OpenStack versions starting with OpenStack Train, in their
  appropriate cloud archives, or PPAs.

- Power drivers in Ironic, require dependencies to be satisfied. Power driver
  dependencies are listed in `driver-requirements.txt <https://github.com/ope
  nstack/ironic/blob/master/driver-requirements.txt>`_

  - Library versions for power drivers must be made available either in the
    cloud archive for the OpenStack version that is being installed, or in a
    separate PPA.
