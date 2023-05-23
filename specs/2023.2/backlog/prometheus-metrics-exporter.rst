..
  Copyright 2021 Canonical Ltd

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===================================================
Align Prometheus metrics exporters to any LMA stack
===================================================

The main objective of this spec is to improve the observability of bind
and haproxy across the OpenStack Charms using Prometheus that is a popular
tool for metrics gathering. Currently, OpenStack charms (except Ceph) do not
support metrics gathering on their components.

Problem Description
===================

The goal of this spec is to standardize the Juju interfaces required
(and their logic) on OpenStack charms in order to facilitate
Prometheus metrics collection (via Prometheus charms) and dashboards
exports (via Grafana charms), no matter which LMA stack is used.

The alert rules that will be configured on prometheus for monitoring bind and
haproxy are not part of this scope.

Proposed Change
===============

Prometheus-scrape interface
---------------------------
This interface is reactivated form the
`prometheus_scrape library <https://github.com/canonical/prometheus-k8s-operator/blob/main/lib/charms/prometheus_k8s/v0/prometheus_scrape.py>`_,
which is used in the new `COS <https://charmhub.io/topics/canonical-observability-stack>`_.
It is written for the Operator framework, so it will be necessary to wrap
`MetricsEndpointConsumer(Object)` to `PrometheusScrapeRequires(Endpoint)` and
`MetricsEndpointProvider(Object)` to `PrometheusScrapeProvides(Endpoint)`.

The interface `charm-interface-prometheus-scrape <https://github.com/openstack-charmers/charm-interface-prometheus-scrape>`_
already implements the `PrometheusScrapeProvides(Endpoint)` part and thus it
can be used to relate reactive charms with `prometheus-k8s-operator <https://github.com/canonical/prometheus-k8s-operator>`_
charm.

Bind
----

BIND service does not support a built-in Prometheus metrics exporter.
However, this service can expose a statistics channel, used by the
bind-exporter to retrieve metrics to Prometheus
(see `bind-exporter service <https://github.com/prometheus-community/bind_exporter>`_).

To take advantage of the bind exporter, changes need to be made in the
charm designate-bind.

Charm designate-bind
^^^^^^^^^^^^^^^^^^^^

This charm needs to expose a statistics service for bind whenever it is related
with the prometheus charm.

The `stats.conf` will have the following content:

.. code-block:: none

  statistics-channels {
    inet <stats-listen-net> port <stats-port> allow { <client-ip>; };
  };

Where stats-listen-net, stats-port and client-ip default values are 127.0.0.1,
8053 and 127.0.0.1, respectively. This will open a statistics channel in bind
that `prometheus_scrape interface` can expose metrics for prometheus to
collect.

This new configuration file will be included at `/etc/bind/named.conf`
appending the following content:

.. code-block:: none

  include "/etc/bind/stats.conf";

When prometheus is removed, it will trigger the reactive endpoint logic for
departed that will be responsible for removing the `stats.conf`, `named.conf`
files and removing it from the restart_map.

**Configuration options**

* no configurations will be available. The charm will automatically configure
  ports and listen network (localhost).


**Relations:**

The charm-designate-bind will support the `prometheus_scrape interface`
to relate directly with prometheus charm and will have all the logic necessary
to install the `snap <https://launchpad.net/prometheus-bind-exporter-snap>`_
responsible for the prometheus bind exporter. The charm will interact with the
snap by setting listen-address and stats-groups.

Furthermore, charm-designate-bind would also relate to the Grafana charm
to share a rendered template with an optimized Grafana dashboard.

Provides:

* **bind-exporter:prometheus_scrape** - The relation connecting this charm with
  the prometheus charm, while providing hostname and port through which it will
  collect all the metrics in Prometheus.
* **grafana:grafana-dashboard** - The relation connecting this charm with the
  Grafana charm and at the same time responsible for creating a new dashboard
  from charm resource.

**Note**: The `render_grafana_dashboard <https://github.com/juju/charm-helpers/blob/bf9c2d83ed579ffb369abbb687fbdf2de62b4d54/charmhelpers/contrib/openstack/ha/utils.py#L353>`_
function from charmhelpers will be responsible for rendering the dashboard.


**Actions**:

This charm requires no action

HAproxy
-------

HAProxy enterprise Edition and from `version 2.0 <https://www.haproxy.com/blog/haproxy-exposes-a-prometheus-metrics-endpoint/>`_
also HAProxy community edition has built-in support to export metrics to
Prometheus via Prometheus exporter.

To take advantage of the Prometheus exporter, changes need to be made in the
`haproxy.cfg` file in the stats section.

.. code-block:: yaml

  listen stats:
      mode http
      stats enable
      bind {{ stats_exporter_host }}:{{ stats_exporter_port }}
      option http-use-htx
      http-request use-service prometheus-exporter if { path /metrics }

To see more information about `stats_exporter_host` and
`stats_exporter_port`, see the Charmhelpers >> `haproxy.cfg` section below.

**Implementation**

For the legacy charms the implementation should be split into five parts:

* editing `haproxy.cfg` config file -  by `charmhelpers` in classic charms
  and by `charms.openstack` for reactive charms (see section below)
* rendering Grafana dashboard template - by `charmhelpers` (see
  section below)
* adding configuration options - by charm itself
* adding and managing relations changes - by charm itself
* providing Grafana dashboard template (JSON) - by charm itself via resource

Charmhelpers
^^^^^^^^^^^^

**haproxy.cfg**

The charmhelpers library implements the code responsible for creating the
context of `haproxy.cfg`, because of that it will be also responsible for
adding parts needed to enable prometheus exporter. The prometheus exporter
will be enabled only if these conditions are met:

* Ubuntu Focal or above (haproxy version >= 2.0)
* Haproxy-exporter relation exists

If both conditions are satisfied, then two values will be needed:

* **stats_exporter_host** - obtain IP address from relation
  (aka. using “get_relation_ip”)
* **stats_exporter_port** - obtain from “haproxy-exporter-stats-port”
  (provided automatically by the charm)

**Note**: In reactive charms this can be done by charms.openstack instead
of charmhelpers.

**Dashboard**

The dashboard template will be provided as a juju resource in a JSON file.
This way it's not necessary to produce a release of charmhelpers library in
order to update the dashboard and gives more flexibility to edit the template
as necessary.

Charms
^^^^^^

The following charms will benefit from these changes, however the list may not
be considered exhaustive at the time of writing.

* charm-ceilometer
* charm-ceph-radosgw
* charm-keystone
* charm-neutron-api
* charm-nova-cloud-controller
* charm-openstack-dashboard
* charm-cinder
* charm-glance
* charm-heat
* charm-swift-proxy
* charm-designate
* charm-neutron-gateway
* charm-percona-cluster
* charm-vault
* charm-ironic-api

**Configuration options**

No configurations will be available. The charm will configured
automatically the port to Prometheus exporter.

**Relations**

Provides:

* **haproxy-exporter:prometheus_scrape** - The relation connecting this charm
  with the Prometheus charm, while providing hostname and port through which it
  will collect all the metrics in Prometheus.
* **grafana:grafana-dashboard** - The relation connecting this charm with the
  Grafana charm and at the same time responsible for creating a new dashboard
  from charm resource.

**Note**: The `render_grafana_dashboard <https://github.com/juju/charm-helpers/blob/bf9c2d83ed579ffb369abbb687fbdf2de62b4d54/charmhelpers/contrib/openstack/ha/utils.py#L353>`_
function from charmhelpers will be responsible for rendering the dashboard.

**Actions**:

No actions required.

References
----------

* https://snapcraft.io/docs/go-plugin
* https://www.haproxy.com/blog/haproxy-exposes-a-prometheus-metrics-endpoint/


Alternatives
------------

None


Implementation
==============

Assignee(s)
-----------

Primary assignee:
- Robert Gildein <robert.gildein@canonical.com>


Gerrit Topic
------------

Use Gerrit topic "prometheus-metrics-exporter" for all patches related
to this spec.

.. code-block:: bash

    git-review -t prometheus-metrics-exporter

Work Items
----------

The Proposed Change and Repositories sections describe the working items.

Repositories
------------

* `prometheus-haproxy-exporter <https://launchpad.net/prometheus-haproxy-exporter-snap>`_
* `prometheus-bind-exporter <https://snapcraft.io/prometheus-bind-exporter>`_
* `charm-interface-prometheus-scrape <https://github.com/openstack-charmers/charm-interface-prometheus-scrape>`_
* `charm-designate-bind <https://opendev.org/openstack/charm-designate-bind>`_
* `charm-helpers <https://github.com/juju/charm-helpers>`_
* `charm-hacluster <https://opendev.org/openstack/charm-hacluster>`_
* `charm-ceilometer <https://opendev.org/openstack/charm-ceilometer>`_
* `charm-ceph-radosgw <https://opendev.org/openstack/charm-ceph-radosgw>`_
* `charm-keystone <https://opendev.org/openstack/charm-keystone>`_
* `charm-neutron-api <https://opendev.org/openstack/charm-neutron-api>`_
* `charm-nova-cloud-controller <https://opendev.org/openstack/charm-nova-cloud-controller>`_
* `charm-openstack-dashboard <https://opendev.org/openstack/charm-openstack-dashboard>`_
* `charm-cinder <https://opendev.org/openstack/charm-cinder>`_
* `charm-glance <https://opendev.org/openstack/charm-glance>`_
* `charm-heat <https://opendev.org/openstack/charm-heat>`_
* `charm-swift-proxy <https://opendev.org/openstack/charm-swift-proxy>`_
* `charm-designate <https://opendev.org/openstack/charm-designate>`_
* `charm-neutron-gateway <https://opendev.org/openstack/charm-neutron-gateway>`_
* `charm-percona-cluster <https://opendev.org/openstack/charm-percona-cluster>`_
* `charm-vault <https://opendev.org/openstack/charm-designate-vault>`_
* `charm-ironic-api <https://opendev.org/openstack/charm-ironic-api>`_

Documentation
-------------

Documentation on how to use prometheus-exporter for HAProxy and
prometheus-bind-exporter for BIND will be necessary.

Security
--------

None

Testing
-------

Code written or changed will be covered by unit tests and functional test.
The functional test will be implemented using the ``Zaza`` framework.

Dependencies
============

None
