..
  Copyright 2021 Canonical Limited.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

========================================
designate-bind: Configurable Service IPs
========================================

Original LP bug: https://bugs.launchpad.net/charm-designate-bind/+bug/1804057

The problem described by the Bootstack team is that when designate-bind [1]
unit get redeployed, there is a chance that they’ll receive a new IP address.
This is a problem if the previous IP address was used in other, non-juju,
parts of the infrastructure to reference the DNS service provided by the
designate (e.g. if the IPs were used in upstream DNS server for zone delegation
or if they were used in forwarding/firewall rules). Suggested solution is to
allow configuration of static IP addresses that would be preserved on the units
even after redeployment.

Problem Description
===================

The possibility of unit’s IP address change requires that each deployment
needs to keep track of all the external systems that reference designate-bind
units by the IP addresses and manually change these references whenever the
unit gets redeployed. These manual actions present an opportunity for human
error and introduce a delay to the update process as, for example, it takes
time for SOA DNS records to propagate depending on their timeout values.

Proposed Change
===============

The core of the solution should be a new config option in designate-bind
charm that would allow to specify static IP addresses from reserved pool not
controlled by the DHCP. This config option would be called service-ips and it
would contain a list of comma separated IP addresses that could be statically
configured on the “DNS frontend” facing interface of the designate-bind units
in addition to the address assigned normally by the DHCP.

There are two alternative solutions suggested in the original launchpad issue
and comments. One Solution would have the designate-bind charm handle the
address assignment and distribution itself, the other suggested solution would
be based on using hacluster subordinate charm [2] that would manage service
IPs as a resource.

Hacluster-based solution offers certain advantages so it’s listed as a main
proposed solution.

Hacluster based solution
------------------------

This solution proposes to use the hacluster subordinate charm. The
service-ips would be configured via `juju config` on the designate-bind charm,
but designate-bind would not be in charge of managing the IPs. Instead, the
relation with charm-hacluster would be used to create each service IP as a
cluster resource in pacemaker [3]. These resources could be configured with
colocation rules such as they would never be placed on the same host if there
was another host without assigned Service IP. This can be achieved by
specifying a negative colocation score for the resources [6] (see score
calculations at [7]).

Another restriction for the Service IP cluster resource might be the actual
network configuration on the designate-bind units. The Service IP must be
within the network configured on the unit and multiple units do not necessarily
need to be on the same network. Cluster resource constraint “location” [9]
can be used to configure cluster resources to prefer or avoid certain hosts.

At the cost of adding load to the designate-bind unit by running
charm-hacluster, this solution is rather elegant and keeps most of the logic
and responsibility out of the charm’s code. Additional benefit is that even in
degraded state, when some of the units are down, all the Service IPs would
remain functional as the hacluster charm would move them to the unit that is
up.


Alternatives
------------

This alternative solution would manage everything within the designate-bind
charm without addition of new dependencies.

As the designate-bind units support HA deployments and are typically deployed
in pairs (e.g. ns1.example.org, ns2.example.org), we need to manage how the
units will choose IP from the service-ips list. This responsibility could be
handled by the leader unit. When the leader unit comes online, it can pick the
first IP address from the list and mark it as “used”. Then, when additional
units join the cluster, the service IP will be chosen for them by the leader,
marked as used, and shared via relationship data. The callback in the leader
unit that chooses the IP address from service-ips, should implement some form
of lock, to prevent collision if two units join the cluster relation at the
same time. Leader unit should use `leader-set` [8] to store information about
“used” IP addresses to ensure preservation of this information in case that
the leader unit changes.

Actual configuration of the additional IP address on each designate-bind unit
could be done using netplan which supports defining static IP on the interface
that has DHCP enabled. It’s done by adding an `addresses` field to the
interface object while also leaving the option `dhcp4: True` [4].

If a designate-bind unit is removed from the cluster, the leader unit should
return the previously assigned IP address to the pool of available service IPs.

In case that more units join the designate-bind cluster than there are
available Service IP addresses, the unit without the service IP should be put
into the blocked state with appropriate message.

In case that the service-ips option is modified and the new number of IP
addresses is less than the current number of deployed designate-bind units, the
leader should redistribute available IP addresses and units without it should
be put into the blocked state with appropriate message.

In case that the service-ips option is changed from having values to empty
value, the leader unit should trigger reconfiguration on all the designate-bind
units that would remove previously statically configured IP addresses. If there
are units that were previously blocked due to the lack of available service
IPs, they should be brought back to the active state.

In case that the service-ips option is not set at all, no special action should
be performed by the leader unit.

Optional: In addition to the configuration change, a confirmatory juju action
could be implemented that would apply the “service-ips” config values. Without
the action, no changes to the live configuration would be made. Reasoning for
this is that it would serve as a confirmation by the user that the change is
intended and it would prevent unwanted changes to the “service-ips”
configuration and possible outages. Another benefit of this action would be
that it could be applied 1 unit at a time so if there was something wrong with
the configuration it would not affect the whole cluster at the same time but
only one unit.

Further Information
-------------------

Regardless of the approach taken, user should only interact with this new
feature by using `juju config` command, for example::

    $ juju config designate-bind service-ips=”10.0.10.10,10.0.10.20”


Basic checks should be performed on the supplied IP addresses:

* Validate that the supplied values are valid IPs
* Validate that the Service IP belongs to the subnet configured on the unit.

References
==========

[1] `<https://jaas.ai/designate-bind/36>`_

[2] `<https://jaas.ai/hacluster>`_

[3] `<https://github.com/openstack/charm-interface-hacluster#requires>`_

[4] `<https://askubuntu.com/questions/1031278/does-netplan-support-dhcp-and-static-addresses-on-one-interface>`_

[5] `<https://github.com/openstack/charm-designate-bind/blob/master/src/metadata.yaml#L23>`_

[6] `<https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/high_availability_add-on_reference/s1-colocationconstraints-haar>`_

[7] `<https://www.thegeekdiary.com/managing-resource-startup-order-in-pacemaker-cluster-managing-constraints/>`_

[8] `<https://charm-helpers.readthedocs.io/en/latest/api/charmhelpers.core.hookenv.html#charmhelpers.core.hookenv.leader_set>`_

[9] `<https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/high_availability_add-on_reference/ch-resourceconstraints-haar>`_

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  martin-kalcok <martin.kalcok@canonical.com>

Gerrit Topic
------------

Use Gerrit topic "<topic_name>" for all patches related to this spec.

.. code-block:: bash

    git-review -t designate-bind-serivce-ips

Work Items
----------

* Add configuration option `service-ips`
* Handler for `service-ips` config change event that configures apropriate
  IP addresses on designate-bind units

Repositories
------------

Work will be located in the main designate bind repository:

`<https://opendev.org/openstack/charm-designate-bind>`_

Documentation
-------------

`To Be Updated`

Security
--------

None

Testing
-------

Unit and Functional tests will be needed to verify this new functionality.

Dependencies
============

None

