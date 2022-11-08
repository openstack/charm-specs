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

======================================================================
Pausing Charms with subordinate hacluster without sending false alerts
======================================================================

Overall, the goal is to leave “warning” alerts instead of “critical” that
will help a human operator understand that all services are not completely
healthy while reducing the criticality due to an on-going operation. Nrpe
checks will be reconfigured once services under a maintenance operation
are set back to normal (resume).

The following logic will be applied when pausing/resuming a unit:

- Pausing a principal unit, pauses the subordinate hacluster;
- Resuming a principal unit, resumes the subordinate hacluster;
- Pausing a hacluster unit,  pauses the principal unit;
- Resuming a hacluster unit, resumes the principal unit;


Problem Description
===================

We need to stop sending false alerts when a hacluster subordinate of an
Openstack charm unit is paused or when the principal unit is also paused
for maintenance. This may help operations to receive more effective alerts.

There are several charms that use hacluster and NRPE that may benefit from
this:

- charm-ceilometer
- charm-ceph-radosgw
- charm-designate
- charm-keystone
- charm-neutron-api
- charm-nova-cloud-controller
- charm-openstack-dashboard
- charm-cinder
- charm-glance
- charm-heat
- charm-swift-proxy


Pausing Principal Unit
----------------------
If eg. 3 keystone units (keystone/0, keystone/1 and keystone/2) are deployed
and keystone/0 is paused:

1) haproxy_servers on the other units (keystone/1 and keystone/2) will alert,
because apache2 service on keystone/0 is down

2) haproxy, apache2.service and memcached.service in keystone/0 will also alert

3) it's possible that corosync and pacemaker have the VIP placed on the same
unit at which point the service will fail as haproxy is disabled. So hacluster
subordinate unit should also be paused.

Note: Services affected when pausing a principal unit may change depending on
the principal charm

Pausing hacluster unit
----------------------

Pausing hacluster set the cluster node, e.g keystone, in standby mode.
A standby node will have its resources stopped (hacluster, apache2) which will
fire false alerts. To solve this issue, the units of hacluster should inform
the keystone unit that they are paused. A way of doing this is through the ha
relation.


Proposed Change
===============

Pausing Pausing Principal Unit
------------------------------
Pause action on a principal unit should share the event with its peers to
modify the behavior on them (until the resume action is triggered).  It should
also share the status (paused/resumed) to the subordinate unit to be able to
catch-up the same status.

File actions.py in the principal unit

.. code-block:: python

  def pause(args):
      pause_unit_helper(register_configs())

      # Logic added to share the event with peers
      inform_peers_if_ready(check_api_unit_ready)
      if is_nrpe_joined():
        update_nrpe_config()

      # logic added to inform hacluster subordinate unit has been paused
      relid = relation_ids('ha')
      for r_id in relid:
          relation_set(relation_id=r_id, paused=True)

  def resume(args):
      resume_unit_helper(register_configs())

      # Logic added to share the event with peers
      inform_peers_if_ready(check_api_unit_ready)
      if is_nrpe_joined():
        update_nrpe_config()

      # logic added to inform hacluster subordinate unit has been resumed
      relid = relation_ids('ha')
      for r_id in relid:
          relation_set(relation_id=r_id, paused=False)

After pausing a principal unit, it will change the unit-state-{unit_name}
to NOTREADY. E.g:

.. code-block:: yaml

  juju show-unit keystone/0 --endpoint cluster
  keystone/0:
    workload-version: 17.0.0
    machine: "1"
    opened-ports:
    - 5000/tcp
    public-address: 10.5.2.64
    charm: cs:~openstack-charmers-next/keystone-562
    leader: true
    relation-info:
    - endpoint: cluster
      related-endpoint: cluster
      application-data: {}
      local-unit:
        in-scope: true
        data:
          admin-address: 10.5.2.64
          egress-subnets: 10.5.2.64/32
          ingress-address: 10.5.2.64
          internal-address: 10.5.2.64
          private-address: 10.5.2.64
          public-address: 10.5.2.64
          unit-state-keystone-0: NOTREADY

Note: unit-state-{unit_name} field is already implemented, I’m just proposing
to use this field and change the value to NOTREADY when a unit is paused and
return to READY when resumed.


With every unit knowing which one is paused, it is possible to change the
script check_haproxy.sh to accept a flag to warn the keystone units that
are paused. The bash script is not able now to receive flags.

Check_haproxy.sh could be rewritten from Bash to Python to accept a flag
to warn specific hostname (e.g. check_haproxy.py --warning keystone-0) is
under maintenance.

The file nrpe.py on charmhelpers/contrib/charmsupport should have changes
to first check if there is any paused unit in the cluster and then add the
warning flag if necessary

.. code-block:: python

  def add_haproxy_checks(nrpe, unit_name):
      """
      Add checks for each service in list

      :param NRPE nrpe: NRPE object to add check to
      :param str unit_name: Unit name to use in check description
      """
      cmd = "check_haproxy.py"

      peers_states = get_peers_unit_state()
      units_not_ready = [
          unit.replace('/', '-')
          for unit, state in peers_states.items()
          if state == UNIT_NOTREADY
      ]

      if is_unit_paused_set():
          units_not_ready.append(local_unit().replace('/', '-'))

      if units_not_ready:
          cmd += " --warning {}".format(','.join(units_not_ready))

      nrpe.add_check(
          shortname='haproxy_servers',
          description='Check HAProxy {%s}' % unit_name,
          check_cmd=cmd)
      nrpe.add_check(
          shortname='haproxy_queue',
          description='Check HAProxy queue depth {%s}' % unit_name,
          check_cmd='check_haproxy_queue_depth.sh')

When a principal unit changes the state e.g READY to NOTREADY, it’s necessary
to rewrite the nrpe files on the other principal units in the cluster because,
otherwise, they won’t be able to warn that a unit is under maintenance.

File responsible for hooks in the classic charms:

.. code-block:: python

  @hooks.hook('cluster-relation-changed')
  @restart_on_change(restart_map(), stopstart=True)
  def cluster_changed():
      # logic added to update nrpe_config in all principal units when
      # a status is changed
      update_nrpe_config()

Note: In reactive charms, it might be slightly different using handlers, but
the mean idea is to update_nrpe_config every time that a config in the cluster
is changed. This will prevent false alerts in the other units in the cluster.


Services from Principal Unit
------------------------------

Removing the .cfg files, when the unit is paused, for those services at
/etc/nagios/nrpe.d would stop sending critical errors. The downside of this
approach is that it won’t have user friendly messages in Nagios saying that
the specific services (apache2, memcached and etc) is under maintenance, on
the other hand, it’s simpler to be achieved.

File responsible for hooks in a classic charm:

.. code-block:: python

  @hooks.hook('nrpe-external-master-relation-joined',
              'nrpe-external-master-relation-changed')
  def update_nrpe_config():
      # logic before change
      # ...

      nrpe_setup = nrpe.NRPE(hostname=hostname)
      nrpe.copy_nrpe_checks()

      # added logic to remove services
      if is_unit_paused_set():
          nrpe.remove_init_service_checks(
              nrpe_setup,
              _services,
              current_unit
          )

      else:
          nrpe.add_init_service_checks(
              nrpe_setup,
              _services,
              current_unit
          )

      # end of added logic

      nrpe.add_haproxy_checks(nrpe_setup, current_unit)
      nrpe_setup.write()

The new logic to remove those services is presented below.

File charmhelpers/contrib/charmsupport/nrpe.py

.. code-block:: python

  # added logic to remove apache2, memcached and etc...
  def remove_init_service_checks(nrpe, services, unit_name):
      for svc in services:
          if host.init_is_systemd(service_name=svc):
              nrpe.remove_check(
                  shortname=svc,
                  description='process check {%s}' % unit_name,
                  check_cmd='check_systemd.py %s' % svc
              )

The status of the services will disappear from nagios after some minutes.
When the resume action is used, the services are restored initially as
PENDING, but after some minutes the check is done.

Pausing hacluster unit
----------------------

File actions.py in charm-hacluster:

.. code-block:: python

  def pause(args):
      """Pause the hacluster services.
      @raises Exception should the service fail to stop.
      """
      pause_unit()
      # logic added to inform keystone that unit has been paused
      relid = relation_ids('ha')
      for r_id in relid:
          relation_set(relation_id=r_id, paused=True)


  def resume(args):
      """Resume the hacluster services.
      @raises Exception should the service fail to start."""
      resume_unit()
      # logic added to inform keystone that unit has been resumed
      relid = relation_ids('ha')
      for r_id in relid:
          relation_set(relation_id=r_id, paused=False)


Pausing a hacluster would result in sharing a new variable paused that can be
used in the principal units.


File responsible for hooks in a classic charm:

.. code-block:: python

  @hooks.hook('ha-relation-changed')
  @restart_on_change(restart_map(), restart_functions=restart_function_map())
  def ha_changed():

      # Added logic to pause keystone unit when hacluster is paused
      for rid in relation_ids('ha'):
          for unit in related_units(rid):
              paused = relation_get('paused', rid=rid, unit=unit)
              clustered = relation_get('clustered', rid=rid, unit=unit)
              if clustered and is_db_ready():
                  if paused == 'True':
                      pause_unit_helper(register_configs())

                  elif paused == 'False':
                      resume_unit_helper(register_configs())

                  update_nrpe_config()
                  inform_peers_if_ready(check_api_unit_ready)
                  # inform subordinate unit that is paused or resumed
                  relation_set(relation_id=rid, paused=is_unit_paused_set())

By informing peers and updating the nrpe config this will be enough to trigger
the necessary logic to remove the services checks.

In a situation where the principal unit is paused, hacluster should also be
paused. For this to happen, it can use the ha-relation-changed from
charm-ha-cluster:

.. code-block:: python

  @hooks.hook('ha-relation-joined',
              'ha-relation-changed',
              'peer-availability-relation-joined',
              'peer-availability-relation-changed',
              'pacemaker-remote-relation-changed')
  def ha_relation_changed():
      # Inserted logic
      # pauses if the principal unit is paused
      paused = relation_get('paused')
      if paused == 'True':
          pause_unit()
      elif paused == 'False':
          resume_unit()

      # share if the subordinate unit status
      for rel_id in relation_ids('ha'):
          relation_set(
              relation_id=rel_id,
              clustered="yes",
              paused=is_unit_paused_set()
          )

Alternatives
------------
One alternative to services from the principal unit checks is to change
systemd.py in charm-nrpe to accept flag -w like the proposal for the
check_haproxy.py

This way would not be necessary to remove the .cfg files for services from
the principal unit, but would be necessary to adapt the function
`add_init_service_checks` to be able to accept services with the warning flag.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  gabrielcocenza

Gerrit Topic
------------

Use Gerrit topic "pausing-charms-hacluster-no-false-alerts" for all patches
related to this spec.

.. code-block:: bash

    git-review -t pausing-charms-hacluster-no-false-alerts

Work Items
----------
- charmhelpers

  - nrpe.py

  - check_haproxy.py

- charm-ceilometer
- charm-ceph-radosgw
- charm-designate
- charm-keystone
- charm-neutron-api
- charm-nova-cloud-controller
- charm-openstack-dashboard
- charm-cinder
- charm-glance
- charm-heat
- charm-swift-proxy

- charm-nrpe (Alternative)

  - systemd.py

- charm-hacluster

  - actions.py

Repositories
------------

No new git Repository is required.

Documentation
-------------

It will be necessary to document the impact of pausing/resuming a
subordinate hacluster and the side effect on Openstack API charms.

Security
--------

No additional security concerns.

Testing
-------

Code changes will be covered by unit and functional tests. For functional
tests, it will use a bundle with keystone, hacluster, nrpe and nagios.

Dependencies
============
None
