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

=======================
Internal DNS Resolution
=======================

Users of an OpenStack cloud would like to look up their instances by name in an
intuitive way using the Domain Name System (DNS). When booting an instance
using Nova, a name is provided which is then used as the hostname for the
system. Many programs in the booted instance utilize DNS to resolve the
hostname and function less than ideally when the hostname cannot be resolved.

Problem Description
===================

The lack of sane internal DNS resolution has plagued users of OpenStack
deployed instances since before the ice(-house) age. Neutron has long provided
DNS services for tenant networks, but the default names provided by Neutron are
defined as 'host-%(octet0)s-%(octet1)s-%(octet2)s-%(octet3)s.openstacklocal'.
Since this name does not match the instance name/hostname, the DNS queries fail
when attempting to resolve the hostname.

Some examples where this becomes problematic for tenants are as follows:

#. Sudo attempts to resolve the hostname each time it runs. Sudo continues to
   work, but complains about the failed name resolution::

    ubuntu@john-oliver:~$ sudo -s
    sudo: unable to resolve host john-oliver

#. Users are unable to logically refer to other instances by name in normal
   network operations as they would in a bare metal lab environment::

    ubuntu@jon-stewart:~$ ping john-oliver
    ping: unknown host john-oliver

#. Applications such as rabbitmq-server require a resolvable hostname in order
   to function properly.

OpenStack Neutron and Nova projects have previously addressed this in the
Liberty and Mitaka releases respectively and the charms should enable this
functionality for the end users.

Proposed Change
===============

The Liberty release of Neutron introduced the DNS extension driver [#]_. This
extension driver, when enabled, adds the ``dns_domain`` attribute to the
Neutron Network API and added the ``dns_name`` and ``dns_fqdn`` (read-only)
attributes to the Neutron Port API. These new attributes are used by the
Neutron dhcp-agent when creating the host file for the dnsmasq process.

.. [#] https://specs.openstack.org/openstack/neutron-specs/specs/liberty/internal-dns-resolution.html

In the subsequent Mitaka release, the Nova project leveraged the newly minted
DNS extension in Neutron to supply DNS-sanitized versions of the instance
name [#]_. When booting new instances, Nova will provide the ``dns_name``
attribute for ports it creates via the Neutron API.

.. [#] https://specs.openstack.org/openstack/nova-specs/specs/mitaka/implemented/internal-dns-resolution.html

OpenStack documentation [#]_ in the Mitaka release provides details for how to
enable this DNS integration between the two projects. The configuration pieces
entirely belong to the Neutron project as Nova will Just Do the Right Thing
(TM).

.. [#] https://docs.openstack.org/mitaka/networking-guide/config-dns-int.html

This spec proposes to enable internal DNS resolution for deployed clouds and
does not attempt to cover integration with an external DNS service such as
Designate.

It is important to note that the internal DNS resolution provided by this
feature is *network isolated* rather than tenant isolated. As such, instances
launched in network A will not be able to resolve instances launched in network
B. For this type of name resolution, an external DNS service such as those
provided by Designate should be used instead and is beyond the scope of this
spec.

To enable the internal DNS resolution, the neutron-server service must be
configured with the ``dns_domain`` config option set to a value other than
``openstacklocal`` in neutron.conf file and the ``dns`` extension driver must
be enabled in the ml2_conf.ini file.

This change will add a two new config options as defined below:

.. code-block:: yaml

    enable-ml2-dns:
      type: boolean
      default: False
      description: |
        Enables the Neutron DNS extension driver. When enabled, ports attached
        to Nova instances will have DNS names assigned based on the instance
        name.
    dns-domain:
      type: string
      default: openstack.example.
      description: |
        Specifies the dns domain name that should be used for building instance
        hostnames. An empty option or the value of 'openstacklocal' will cause
        the dhcp agents to broadcast the default domain of openstacklocal and
        will not enable internal cloud dns resolution. This value should end
        with a '.', e.g. 'cloud.example.org.'.

In order to enable internal DNS resolution, the user must set the
``enable-ml2-dns`` to True. The default value is False in order to provide
backwards compatibility with existing deployments.

Relation Implications
---------------------

The neutron-dhcp-agent does not run on the same node as the neutron-server and
should have the ``dns_domain`` specified in the dhcp_conf.ini file. Specifying
this value in the dhcp_conf.ini file is not strictly necessary, as the
hostnames will properly resolve without it. However, the default search list
advertised to hosts will be the default ``openstacklocal`` and may cause
confusion to users. To allow the neutron-api charm to share this configuration
information with interested parties, the ``neutron-plugin-api`` relation-data
will be updated to contain the dns-domain name:

.. code-block:: yaml

    'dns-domain': 'domain.tld.'

The value will be a string containing the domain name.

Alternatives
------------

#. Designate could be setup to provide the DNS service entries for the tenant.
   This option is valid, but requires additional components to be setup and
   deployed into the environment. Additionally, there are some limitations
   which are not well documented in the upstream documentation for configuring
   DNS integration. For example, the Neutron port API will not call the
   Designate API for ports on tunnelled tenant networks (e.g. GRE).

#. An out-of-band solution such as that provided by the serverstack-dns
   tool could be deployed to provide DNS based upon OpenStack events. This is
   less than desirable as it must be installed per tenant, each tenant must
   have access credentials to access the resources of the underlying cloud, and
   the tool itself is not intended for a production environment.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  billy-olsen


Gerrit Topic
------------

Use Gerrit topic "charms-internal-dns" for all patches related to this spec.

.. code-block:: bash

    git-review -t charms-internal-dns

Work Items
----------

charm-neutron-api
  Add new config option to the neutron api charm
  Add dns-domain to the neutron-plugin-api interface
  Update README.md to reflect new behavior

charm-neutron-gateway
  Update neutron-gatway to consume dns-domain from relation data

Repositories
------------

No new git repositories required.

Supported Releases
------------------

This feature will be available on deployed clouds running Mitaka or newer.
Attempting to enable this feature on earlier versions will have no effect.

Documentation
-------------

The neutron-api charm README will need to be updated to reflect the new feature
and how to enable internal DNS.

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

No external dependencies.
