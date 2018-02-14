..
  Copyright 2018 Canonical UK Ltd

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

========
Cells V2
========

Nova cells v2 has been introduced over the Ocata and Pike cycles. In fact, all
Pike deployments are now deployments using nova cells v2 usually using a single
compute cell.

Problem Description
===================

Nova cells v2 allows for a group of compute nodes to have their own dedicated
database, message queue and conductor while still being administered through
a central API service. This has the following benefits:

Reduced pressure on Rabbit and MySQL in large deployments
---------------------------------------------------------

In even moderately sized clouds the database and message broker can quickly
become a bottle neck. Cells can be used to alleviate that pressure by having a
database and message queue per cell of compute nodes. It is worth noting that
the charms already support having traffic for neutron etc in a separate rabbit
instance.

Create multiple failure domains
-------------------------------

Grouping compute cells with their local services allows the creation of
discrete failure domains (from a nova POV at least).

Remote Compute cells (Edge computing)
-------------------------------------

In some deployments a group of compute nodes maybe far removed (from a
networking pov) from the central services. In this case it maybe useful to have
the compute nodes act as a largely independent group.

Different SLAs per cell
-----------------------

Different groups of compute nodes can have different levels of performance,
HA. etc. A cell could have no local HA for the database, message queue and
conductor for a development cell but the production cell could have
significantly higher specification servers running clustered services.

(These use cases were paraphrased from `*4 <https://www.openstack.org/videos/sydney-2017/adding-cellsv2-to-your-existing-nova-deployment>`_.)

Proposed Change
===============

To facilitate a cells v2 deployment a few relatively simple interfaces and
relations need to be added. From a nova perspective the topology looks like `this <https://docs.openstack.org/nova/latest/_images/graphviz-d1099235724e647ca447c7bd6bf703c607ddf68f.png>`_.
This spec proposes mapping that to this `charm topology <https://docs.google.com/drawings/d/1v5f8ow0aCGrKRIpg3uXsv2zolWsz3mGVGzLnbgUQpKQ/>`_.

Superconductor access to Child cells
------------------------------------

The superconductor needs to be able to query the databases of the compute cells
and to send and receive messages on the compute_cells message bus. The
cleanest way to model this would be to have a direct Juju relation between the
superconductor and the compute cells database and message bus. To facilitate
this the following relations will be added to the nova-cloud-controller charm:

.. code-block:: yaml

  requires:
    shared-db-cell:
      interface: mysql-shared
    amqp-cell:
      interface: rabbitmq


Superconductor configuring child cells
--------------------------------------

With the above change the superconductor has access to the child db and mq but
does not know which compute cell name to associate with them. To solve this the
nova-cloud-controller charm will have the following new relations:


.. code-block:: yaml

  provides:
    nova-cell-api:
      interface: cell
  requires:
    nova-cell:
      interface: cell

The new cell relation will be used to pass the cell name, db service name and
message queue service name.

.. code-block:: python

   {
     'amqp-service': 'rabbitmq-server-cell1',
     'db-service': 'mysql-cell1',
     'cell-name': 'cell1',
   }

Given this information the superconductor can examine the service names that
are attached to its shared-db-cell and amqp-cell relations and construct
urls for them. The superconductor is then able to create the cell mapping in
the api database by running:

.. code-block:: bash

    nova-manage cell_v2 create_cell \
        --name <cell_name> \
        --transport-url <transport_url> \
        --database_connection <database_connection>

The superconductor needs five relations to be in place and their corresponding
contexts to be complete before the cell can be mapped. Given the
nova-cloud-controller is a non-reactive charm special care will be needed to
ensure that the cell mapping happens irrespective of the order in which those
relations are completed.

Compute conductor no longer registering with keystone
-----------------------------------------------------

The compute conductor does not need to register an endpoint with keystone nor
does it need service credentials. As such the identity-service relation should
not be used for compute cells. A guard should be put in place in the
nova-cloud-controller charm to prevent a compute cells nova-cloud-controller
from registering an incorrect endpoint in keystone.

Compute conductor cell name config option
-----------------------------------------

The compute conductor needs to know its own cell name so that it can pass this
information up to the superconductor. To allow this a new configuration option
will be added to the nova-compute charm:

.. code-block:: yaml

  options:
    cell-name:
      type: string
      default:
      description: |
        Name of the compute cell this controller is associated with. If this is
        left unset or set to api then it is assumed that this controller will
        be the top level api and cell0 controller.

Leaving the cell name unset assumes the current behaviour of associating the
nova-cloud-controller with the api service, cell0 and cell1.

nova-compute service credentials
--------------------------------

The nova-compute charm needs service credentials for RPC calls to the Nova
Placement API and the Neutron API service. It currently gets these credentials
via its cloud-compute relation which is ugly at best. However, given that the
compute cells nova-cloud-controller will no longer have a relation with
keystone it will not have any credentials to pass on to nova-compute. This is
overcome by adding a cloud-credentials relation to the nova-compute charm.

.. code-block:: yaml

  requires:
    cloud-credentials:
      interface: keystone-credentials

nova-compute will request a username based on its service name so that users
for different cells can be distinguished from one another.

Bespoke vhosts and db names
---------------------------

The ability to specify a nova db name and a rabbitmq vhost name should either
be removed from the nova-cloud-controller charm or the new cell interface needs
to support passing those up to the superconductor so that the superconductor
can request access to the correct resources from the compute nodes database and
message queue.

Disabling unused services
-------------------------

The compute cells nova-cloud-controller only needs to run the conductor service
and possible the console services. Unused services should be disabled by the
charm.

New cell conductor charm?
-------------------------

The nova cloud controller in a compute node only runs a small subset of the
nova services and does not require a lot of the complexity that is baked
into the current nova-cloud-controller charm. This begs the question of whether
a new cut-down reactive charm that just runs the conductor would make sense.
Most of the changes outlined above actually impact the superconductor rather
than the compute conductor. However, looking at this the other way around the
changes needed to allow the nova-cloud-controller charm to act as a child
conductor are actually quite small and so probably do not warrant the creation
of a new charm. It is probably worth noting some historical context here too,
every time the decision has been made to create a charm which can operate in
multiple modes that decision has been reversed at some cost at a later data (
ceph being a prime example).

Taking all that into consideration a new charm will not be written and the
existing nova-cloud-controller charm will be extended to add support for
running as a compute conductor.

Message Queues
--------------

There is flexibility around which message queue the non-nova services use. A
dedicated rabbit instance could be created for them or they could reuse the
rabbit instance the nova api service is using.

Telemetry etc
--------------

This spec does not touch on integration with telemetry. However, this does
require further investigation to ensure that message data can be collected.

Juju service names
------------------

It will be useful, but not required, to embed the cell name in the service name
of each component that is cell specific. Eg deploying services for cellN
may look like this:


.. code-block:: yaml

    juju deploy nova-compute nova-compute-cellN
    juju deploy nova-cloud-controller nova-cloud-controller-cellN
    juju deploy mysql mysql-cellN
    juju deploy rabbitmq-server rabbitmq-server-cellN


Alternatives
------------

* Do nothing and do not support additional nova v2 cells.
* Resurrect support for the deprecated and bug ridden cells v1

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Unknown

Gerrit Topic
------------

Use Gerrit topic "<topic_name>" for all patches related to this spec.

.. code-block:: bash

    git-review -t cellsv2

Existing Work
-------------

As part of writing the spec prototype charms and a bundle were created
for reference: `Bundle <https://gist.github.com/gnuoy/9ede4e9d426ea56951c664569e7ad957>`_
and `charm diffs <https://gist.github.com/gnuoy/aff86d0ad616a890ba731a3cb7deef51>`_

Work Items
----------

* Remove support for cells v1 from nova-compute and nova-cloud-controller
  charms
* Add identity-context relation to nova-compute and ensure the supplied
  credentials are used when rendering placement and keystone sections in
  nova.conf
* Add shared-db-cell relation to nova-cloud-controller assuming 'nova'
  database name when requesting access.
* Add amqp-cell relations to nova-cloud-controller assuming 'openstack' vhost
  name when requesting access.
* Add code for registering a  cell to nova-cloud-controller. This will use the
  AMQ and SharedDB contexts from the shared-db-cell and amqp-cell relation
  to create the cell mapping.
* Update nova.conf templates in nova-cloud-controller to only render api db
  url if the nova-cloud-controller is a superconductor.
* Update db initialisation code to only run the relevant cell migration if not
  a superconductor.
* Add nova-cell and nova-cell-api relations and ensure that the shared-db, amqp
  shared-db-cell, amqp-cell and nova-api-cell relations all attempt to register
  compute cells.
* Write bundles to use cells topology
* Check integration with other services (designate and telemetry in particular)

Repositories
------------

No new repositories needed.

Documentation
-------------

* READMEs of nova-cloud-controller and nova-compute will need updating to
  explain new relations and config options.
* Blog with deployment walkthrough and explanation.
* Update Openstack Charm documentation to explain how to do a multi-cell
  deployment
* Add bundle to charm store.

Security
--------

No new security risks that I am aware of

Testing
-------

* A multi-cell topology is probably beyond the scope of amulet tests
* Bundles added to openstack-charm-testing
* Mojo specs

Dependencies
============

None that I can think of

Credits
-------

Much of the benefit of cells etc was lifted from \*4

\*1 https://docs.openstack.org/nova/pike/cli/nova-manage.html
\*2 https://docs.openstack.org/nova/latest/user/cellsv2-layout.html
\*3 https://bugs.launchpad.net/nova/+bug/1742421
\*4 https://www.openstack.org/videos/sydney-2017/adding-cellsv2-to-your-existing-nova-deployment
