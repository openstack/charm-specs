..
  Copyright 2019 Canonical Limited

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

=============================================
Policy overrides using the policy.d directory
=============================================

The objective is to provide a mechanism where operators can modify the policies
of OpenStack services in order to manage the permissions of the tenants and
users of the system.  It is NOT intended that the facility described in this
specification is used to manage the permissions of the services themselves.
This must remain wholly the domain of the charms, and the facility described
here should not be (ab)used.

Problem Description
===================

The preferred approach has always been to use charm options to enable a more
controlled approach to modifying the policy of a service by providing templates
that the options modify - i.e. a controlled approach that always results in
consistent policy files.

However, over time, it has become apparent that there will always be an aspect
of the policy of a service that an operator wants to tweak or change that the
charm authors hadn't considered, and the cycle time to introduce the option
exceeds the expectations of the delay that an operator is willing to tolerate.

Therefore, it has been recognised that there are aspects of the configuration
files that the charms control need to be more fluidly modifiable by operators.
These tend to be orthogonal to the act of deployment and instead are to do with
operating the cloud.

So, the objective is to provide to operators with a mechanism to override the
policy defaults without (hopefully) breaking the system functioning. The policy
defaults are either coded in the service via “policy-in-code” and/or via a
default policy YAML file provided by the charm/service.

Proposed Change
===============

- This functionality will be supported on Ubuntu Xenial and later.
- This functionality will be supported on OpenStack queens and later.
- Juju version 2.0 or later is required for resource support.

The addition of *policy overrides* needs to be provided in the set of target
charms (see later).  In order to do this efficiently most of the work needs to
be performed in *charm-helpers* with minimal work done in each charm.

A config option boolean ``use-policyd-override`` (default ``False``) will
control whether the resource file is used.  If ``False`` then the resource file
is ignored - i.e. it doesn't matter whether it is present/damaged or not.

If config option ``use-policyd-override`` is ``True`` then the Juju status will
be prefixed with ``"PO:"`` indicating that the policies are overridden by the
attached resource file.  This is intentionally kept short so that other
information can also be seen on the status line.  A info log is also generated
to the Juju log to indicate that policies are overridden.

Juju resources will be used to attach a zipped file (using pkzip or equivalent)
to the juju application using the resource name *policy-override*.  The
charm(s) associated with the application will verify the contents of the
zipped file and update the contents of the policy directory (default
``/etc/<service-name>/policy.d`` with the files in the zipped resource file.

The resource name will be ``policyd-override``.

Conceptually, the charm *owns* the ``/etc/<service-name>/policy.d/``.
Subordinate charms will **not** have the facility to override policies; this
should be performed in the principal charm.  The consequence of this
'ownership' is that the charm *will* remove files from the directory if they
are not put there by the charm itself (for its own override purposes) or via
this policy override system.  i.e. *ad hoc* dropping of files into the
directory will most likely be removed during the next ``upgrade-charm`` hook
leading to unexpected results.  Changes to the
``/etc/<service-name>/policy.d/`` directory are logged to the charm's debug
log.

Templates in the Resource File
------------------------------

The policy override feature essentially comes in two parts; charm-helpers (and
``charms.openstack``) and calling the support functions from the charm.  One
slightly contentious issue is around using templates to allow context variables
(built from the relations' data and config in a charm) to be baked into
a policy override file prior to being dropped into the ``policy.d`` directory.

This is not the author's preferred mechanism which is to instead use the
existing template facility to render rules into the charm's default policy file
(in the same was as ``admin_required`` is done in the keystone ``policy.json``
file) and then refer to these in the stack override files.  That is, templates
are still controlled in the charm and not in the drop-in overrides.

However, it is fairly trivial to *offer* a string substitution capability in
the ``charm-helpers`` functions, even if this is not *used* when called by the
charm.  The author believes that this should be the default condition, and only
allow the template facility to be enabled when a policy override requirement
cannot be satisfied in any other way.

Template and substitution will offered so that files in the resource file with
either ``.j2``, ``.tmpl`` or ``.tpl`` extensions are processed by a charm
provided string substitution function which may be a Jinja template function
using the context, or just a simple text substitution with some defined
variables.  This will be under the control of the charm.  The substituted
string returned by the charm-supplied function is then processed as per
a static yaml file with all the following validation.

Validation
----------

The charm will attempt some validation of the attached resource file:

- That the file is a valid zip file (note that any directories are ignored.
  The zip file is flattened to remove directories.
- Files that don't have a ``.yaml`` or ``.yml`` extension are ignored.  This allows
  documentation to be added to the zip file.
- The remaining, flattened, identified yaml files are checked to have unique
  names.  If not, then the validation fails.
- Each file is (attempted to be) loaded using the Python yaml ``safe_load``
  function.  If this fails, then validation fails.
- Each of the policy overrides will be checked against a blacklist that the
  charm maintains.  This blacklist will be on a per-charm basis, and will
  initially blacklist changing ``admin_required`` and ``cloud_admin``.  This is
  to ensure that the charms themselves do not get locked out of the OpenStack
  system that they maintain.
- If validation fails then the **whole** policy override fails and the
  charm proceeds as if ``use-policyd-override`` is ``False``.  The
  application/charm(s) active status will show ``"PO (broken): ..."``.
  Note that there will **not** be a hook error.  The rationale is that
  policy overrides are an *operator* facility and the charm attempts to apply
  them, but should only indicate that they have not been applied so that the
  operator can change the policy overrides.  A message will be logged to
  charm's log explaining the validation failure.

If charm validation succeeds then the ``/etc/<service-name>/policy.d/``
directory is cleared (apart from other charm intrinsic policy overrides) and
then new files written to the directory.  The payload service will be
restarted.

OpenStack Service Support
-------------------------

The policy override feature appeared in ``oslo.policy`` for the ocata release, and was picked up by
keystone and nova initially.  In order for it to be supported, the OpenStack service needs to support
the ``Enforcer`` class to be able to make use of the ``policy.d`` override directory (which is the
default).

The following service projects (that have Canonical charms) examined and have
the ``Enforcer`` class at the queens release:

- keystone
- neutron
- nova
- glance
- cinder
- aodh
- designate
- heat
- horizon (openstack-dashboard)
- manila
- octavia
- panko
- trove
- zaqar

The following service projects do not have ``Enforcer`` class support at present (stein):

- swift

Services where it doesn't apply:

- gnocchi
- ceilometer


This work will affect a number of charms.  `Bug#1741723`_ records the charms
that (most likely) need the policy override facility.  The principal ones that
are the scope of this work are:

- cinder
- designate (reactive charm)
- glance
- keystone
- neutron-api
- nova-cloud-controller

The nature of the implementation should make it relatively easy to implement
this on further charms.  Designate is included to provide a proof-of-concept
for reactive charms.

.. _`Bug#1741723`: https://bugs.launchpad.net/charm-keystone/+bug/1741723

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ajkavanagh (irc: tinwood)

Gerrit Topic
------------

Use Gerrit topic "policy-overrides" for all patches related to this spec.

.. code-block:: bash

    git-review -t policy-overrides

Work Items
----------

The changes will be implemented in:

- charm-helpers (contrib.openstack)
- charms.openstack
- individual charms (keystone, glance, *etc.*)

The following work items have been identified with this work:

- charm-helpers:

  - helpers to read zip file, perform validation
  - helpers to update policy.d directory with validated files.
  - helper to call from install and upgrade hooks
  - helper to call from config-changed hook handler to determine whether to
    enable or disable the policy overrides.
  - helper to merge into update-status to determine whether policy is
    overridden or not.

- charms.openstack:

  - Mix-in that calls out to charm-helpers to perform validation and
    update/control of policy.d directory.
  - Method to integrate into install and upgrade hooks to call relevant
    functions in the mix-in.
  - Method to integrate with config-changed hook to determine whether to
    enable or disable the policy overrides (probably call out to
    charm-helpers).

- keystone charm (*in the first instance*):

  - Link into the helper functions in charm-helpers.
  - Functional test to verify that policy can be overridden and that files
    appear in the policy.d directory.

- designate charm (reactive example)

  - Add in mix-in to designate main class, and any required helper calls (it's
    not clear yet how much additional support will be needed).
  - Functional test to verify that policies can be overridden with a file.

Repositories
------------

The following repositories will be affected:

- `charm-helpers <https://github.com/juju/charm-helpers>`_
- `charms.openstack <https://opendev.org/openstack/charms.openstack>`_
- `cinder charm <https://opendev.org/openstack/charm-cinder>`_
- `designate charm <https://opendev.org/openstack/charm-designate>`_
- `glance charm <https://opendev.org/openstack/charm-glance>`_
- `keystone charm <https://opendev.org/openstack/charm-keystone>`_
- `neutron-api charm <https://opendev.org/openstack/charm-neutron-api>`_
- `nova-cloud-controller charm <https://opendev.org/openstack/charm-nova-cloud-controller>`_

And the testing frameworks:

- `zaza <https://github.com/openstack-charmers/zaza>`_
- `zaza-openstack-tests <in://github.com/openstack-charmers/zaza-openstack-tests>`_

Documentation
-------------

The following documents will require an update:

- charm-guide
- deployment-guide
- README in charms affected by change

Security
--------

Although a file is being uploaded to the controller (and thus as a resource to
the charms), the Python zip module is used to examine the contents, and
potential yaml files are read with the YAML module's ``safe_load`` is used to
verify that the contents are yaml.  This is then written to relevant files in
the ``/etc/<service-name>/policy.d/`` directory.  Thus the files in the zip
file aren't copied to the file directory.  It is thus thought that, as no 3rd
party module are used, and no files are directly copied, that there should not
be a security issue with this approach.

Testing
-------

- Unit tests will be added to charm-helpers and charms.openstack
- Functional tests will be added to those charms that support the zaza testing
  framework.  This will verify that the overrides are put into the appropriate
  directory and that the service is restarted.  A service action (e.g. list
  users) will be verified as working, then disabled, and then working again.
  A framework will be put into zaza-openstack-tests to support this.

Actual policy overrides are not checked.  This is the domain of the payload
(service) and outside of the scope of testing for this feature which is simply
to drop appropriately formatted yaml files into the policy.d directory.

Dependencies
============

There are no dependencies.
