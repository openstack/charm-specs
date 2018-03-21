..
  Copyright 2017 Canonical UK

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===============================
Designate - Neutron integration
===============================

Users of an OpenStack cloud would like to use Designate DNS records
auto-generation feature which is already available in the upstream:

https://docs.openstack.org/ocata/networking-guide/config-dns-int.html

Because of the reasons described below, the existing solutions based on
integration of Designate with Nova via notification_topics does not satisfy
their requirements.

Problem Description
===================

With OpenStack Mitaka release there has been a new feature introduced which is
integration of the Compute service and the Networking service with an external
DNSaaS (DNS-as-a-Service) via extension_drivers. This feature allows DNS
records to be automatically generated in an external DNSaaS (i.e. Designate)
when Neutron ports / floating IPs are created.

The feature has become an official standard for Designate - Neutron
integration and it seems to replace the old approach based on Designate - Nova
integration via notificiation_topics. It also solves the key limitation of the
old approach which was lack of support for multiple DNS domains within the
tenant.

In order to use this feature, the following configuration parameters need to
be set on the Neutron Server side:

#. /etc/neutron/neutron.conf

    [default]
    external_dns_driver = designate

    [designate]
    url = <designate public endpoint>
    admin_auth_url = <keystone admin endpoint>
    admin_username = <admin user username>
    admin_password = <admin user password>
    admin_tenant_name = <admin user tenant>
    allow_reverse_dns_lookup = <allow creation of PTR records>
    ipv4_ptr_zone_prefix_size = <IPv4 PTR zone prefix size>       # optional
    ipv6_ptr_zone_prefix_size = <IPv6 PTR zone prefix size>       # optional
    cafile = <path to CA certificate>                             # optional

#. /etc/neutron/plugins/ml2/ml2_conf.ini

    [ml2]
    extension_drivers = dns

However, currently the neutron-api charm does not support any way of setting
up these parameters, except of ml2 extension.

Proposed Change
===============

The proposed change can be split to the following parts:

#. Create new interface (``designate``) to expose public API endpoint of
   Designate service.

#. Create new relation ("external-dns") between designate and neutron-api
   charms to send Designate public endpoint through relation data via the
   "designate" interface. Implement reactive handlers / hooks on both designate
   and neutron-api charms size to support the new relation.

#. Add new config options to neutron-api charm to handle the following
   parameters:

- "allow_reverse_dns_lookup":

  purpose: specifies whether to enable or not the creation of PTR records
  data type: boolean
  valid values: True / False

- "ipv4_ptr_zone_prefix_size":

  purpose: the size in bits of the prefix for the IPv4 reverse lookup zones
  data type: int
  valid values: 0 - 32

- "ipv6_ptr_zone_prefix_size":

  purpose: the size in bits of the prefix for the IPv6 reverse lookup zones
  data type: int
  valid values: 0 - 128

#. Implement new class ("DesignateContext") on neutron-api charm to generate
   the following context based on relation data and config options:

    ctxt = {'enable_designate': <True if "external-dns" relation established>,
            'designate_endpoint': <consumed from "designate" interface>,
            'allow_reverse_dns_lookup': <based on config option>,
            'ipv4_ptr_zone_prefix_size': <based on config option>,
            'ipv6_ptr_zone_prefix_size': <based on config option>}

#. Update any other context classes which may be affected by the change.

#. Update templates to enable the feature if "external-dns" relation is
   established. Remaining parameters will be retrieved from the existing
   contexts.

Alternatives
------------

The designate charm currently supports integration with nova-compute charm for
DNS records auto-generation via notification_topics. This approach, however,
has some limitations which can be summarized as follows:

* It seems to be obsolete starting from OpenStack Mitaka release in favor of
  Designate - Neutron integration via extension_drivers,
* It lacks support for multiple DNS domains within the tenant.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tkurek

Gerrit Topic
------------

Use Gerrit topic designate-neutron for all patches related to this spec.

.. code-block:: bash

    git-review -t charm-designate-neutron

Work Items
----------

Update designate charm
++++++++++++++++++++++

- Update metadata.yaml and layer.yaml files to support new relation and
  interface
- Add new reactive handler to support new relation and interface
- Update config.yaml file to indicate that old options are obsolete
- Update README.md to describe new feature
- Update unit tests

Update neutron-api charm
++++++++++++++++++++++++

- Update metadata.yaml file to support new relation and interface
- Update neutron-api charm to consume "designate_endpoint" from relation data
- Add new config options to the neutron-api charm
- Update template files to implement upstream feature
- Update README.md to describe new feature
- Update unit tests

Provide designate interface
++++++++++++++++++++++++++

- Create new interface to expose public API endpoint of Designate service

Repositories
------------

A new git repository will be required for the designate interface:

.. code-block:: bash

    git://git.openstack.org/openstack/charm-interface-designate

Documentation
-------------

README files of designate and neutron-api charms will need to be updated to
describe the new feature.

Security
--------

No security implications for this change.

Testing
-------

Unit and functional tests of designate and neutron-api charms will need to be
updated to support implementation of the new feature.

Dependencies
============

No external dependencies.
