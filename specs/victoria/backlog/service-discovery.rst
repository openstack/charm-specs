..
  Copyright 2017 Canonical UK Ltd

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=================
Service Discovery
=================

Many optional services may now be deployed as part of an OpenStack Cloud,
with each service having a different optional features that may or may
not be enabled as part of a deployment.

Charms need a way to discover this information so that services can be
correctly configured for the options chosen by the charm user.

Problem Description
===================

Charms need to be able to determine what other services are deployed
within an OpenStack Cloud so that features can be enabled/disabled as
appropriate.

Examples include:

- notifications for ceilometer (really don't want notifications enabled
  when ceilometer is not deployed).

- misc panels within the openstack dashboard (fwaas, lbaas, l3ha, dvr
  etc... for neutron).

- notifications for designate (disable when designate is not deployed).

Services and features of services are determined by the API endpoint
charms that register them into the service catalog via the keystone charm.

Proposed Change
===============

The keystone charm will expose a new provides interface 'cloud-services'
which is a rich-ish description of the services deployed with registered
endpoints.

The identity-service relations would also advertise the same data as the
cloud-services relations so that charms already related to keystone don't
have to double relate (identity-service is a superset of cloud-services).

By default, a registered endpoint of type 'A' will result in service type
'A' being listed as part of the deployed cloud on this interface.

Services may also enrich this data by providing 'features' (optional)
alongside their endpoint registration - these will be exposed on the
cloud-service and identity-service relations.

Data will look something like (populated with real examples - key and
proposed values):

.. code-block:: yaml

    services: ['object-store', 'network', 'volumev2', 'compute',
               'metering', 'image', 'orchestration', 'dns']

Example - advertise features supported by the networking service, allowing
features to be enabled automatically in the dashboard:

.. code-block:: yaml

    network: ['dvr', 'fwaas', 'lbaasv2', 'l3ha']

Example - allow ceilometer to know that the deployed object-store is radosgw
rather than swift:

.. code-block:: yaml

    object-store: ['radosgw']

Values will be parseable as a json/yaml formatted list.

By using the basic primitive of tags, we get alot of flexibility with
type/feature being easily express-able.

Interface will be eventually consistent in clustered deployments -
all keystone units will present the same data.

Alternatives
------------

Each charm could query the keystone service catalog; however this is very
much a point in time check, and the service catalog may change after the
query has been made. In addition the keystone service catalog does not
have details on what optional features each service type may have enabled
and keystone services will be restarted during deployment as clusters
get built out etc.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  james-page

Gerrit Topic
------------

Use Gerrit topic "service-discovery" for all patches related to this spec.

.. code-block:: bash

    git-review -t service-discovery

Work Items
----------

Core (keystone charm):

- Add cloud-services relation to keystone charm
- Add service and feature discover handler to keystone charm
- Update keystone interface to advertise services and features in
  keystone charm.
- Create cloud-services reactive interface

Enablement:

- Update ceilometer charm for radosgw discovery.
- Update openstack-dashboard charm to automatically enable panels
  for deployed services and features.
- Update neutron-api charm for designate discovery.
- Update cinder charm for ceilometer discovery.
- Update glance charm for ceilometer discovery.
- Update neutron-api charm for ceilometer discovery.
- Update radosgw charm to advertise 'radosgw' feature.
- Update neutron-api charm to advertise networking features.

Repositories
------------

No new git repositories required.

Documentation
-------------

This change is internal for use across the OpenStack charms, no documentation
updates are required for end-users.

Security
--------

No security implications for this change.

Testing
-------

Implementation will include unit tests for all new code written; amulet
function tests will be updated to ensure that feature is being implemented
correctly across the charm set.

Dependencies
============

No external dependencies
