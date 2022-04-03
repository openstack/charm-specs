..
  Copyright 2022 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

======================
Virtual TPM Enablement
======================

Increasingly, applications and Operating Systems are using TPM devices to
store secrets. In order to run these application in a virtual machine, it
is necessary to be able to expose a virtual TPM device within the guest.

Problem Description
===================

Guests requiring access to TPMs for secret storage are unable to do so in an
OpenStack Charms deployed cloud.

Proposed Change
===============

Nova is able to provide virtual TPM devices to guests starting in the Victoria
release [1]_, [2]_. TPM devices are provided to libvirt/qemu guests via the
swtpm library.

The ``nova-compute`` charm should be able to install and configure the
necessary libraries for providing emulated TPM devices. It will do so by
default for new installations and installations that upgrade to a version of
the ``nova-compute`` charm which has the feature set enabled. This will cause
the nova-compute service on the local machine to report that it has the
``COMPUTE_SECURITY_TPM_1_2`` and ``COMPUTE_SECURITY_TPM_2_0`` traits.

While the compute nodes will report that they have the necessary traits, new
instances will not have TPM devices attached unless the flavor and/or image has
the appropriate properties configured. It is considered an administrative
decision to determine which images or flavors should have TPM devices enabled
and is out of scope for this implementation.

The above also makes it generally safe to enable by default for users who
upgrade their charms to a version that has this capability enabled. While it
may be safe to enable by default, initial versions will ship with the feature
disabled by default in order to prevent package installation errors on charm
or OpenStack upgrade.

Charm Configuration Options
---------------------------

The following configuration options will be available on the ``nova-compute``
charm:

* A new config option will be introduced in order to enable or disable vTPM
  support::

      enable-vtpm:
        type: boolean
        default: False
        description: |
          Enable emulated Trusted Platform Module support on the hypervisors.
          A key manager, e.g. Barbican, is a required service for this
          capability to be enabled.


Configuration Files
-------------------

The swtpm package in Ubuntu does not use the tss/tss user/group that is the
default for qemu, nova, etc. Instead, the swtpm package configures the
user/group as swtpm/swtpm as the swtpm user does not need the same level of
permissions that the existing tss user has. This requires some additional
changes to configuration files.

Enabling virtual TPM support using OpenStack charms will require the following
configuration files to be updated:

* */etc/libvirt/qemu.conf* - the `swtpm_user` and `swtpm_group` values need to
  be set to the same users that the swtpm software package expects. This will
  cause the qemu configuration file to look as follows::

      ##########################################################################
      # [ WARNING ]
      # Configuration file maintained by Juju. Local changes may be overwritten.
      ##########################################################################

      # File installed by Juju nova-compute charm
      cgroup_device_acl = [
         "/dev/null", "/dev/full", "/dev/zero",
         "/dev/random", "/dev/urandom",
         "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
         "/dev/rtc", "/dev/hpet", "/dev/net/tun",
         "/dev/vfio/vfio",
      ]

      swtpm_user = "swtpm"
      swtpm_group = "swtpm"

* */etc/nova/nova-compute.conf* - similar to the qemu config changes, the
  nova services need to specify which user and group should be configured in
  libvirt for qemu instances. It will also have the global flag for enabled
  or not enabled::

      ##########################################################################
      # [ WARNING ]
      # Configuration file maintained by Juju. Local changes may be overwritten.
      ##########################################################################
      [DEFAULT]
      compute_driver=libvirt.LibvirtDriver
      swtpm_enabled=True
      swtpm_user=swtpm
      swtpm_group=swtpm


Non-Charm Configuration
-----------------------

Enabling vTPM support in the nova compute charm will cause the nova hypervisor
to report the ``COMPUTE_SECURITY_TPM_1_2`` and ``COMPUTE_SECURITY_2_0``
traits to the placement service. Additional steps need to be taken by the
cloud operator/administrator in order to make this feature available to guests
by configuring the appropriate properties on images or flavors.

Nova uses information from the extra specs configured on the flavor or
properties set on an image in order to determine whether or not to add a vTPM
device. As such, the hypervisor may be configured to have the necessary
traits exposed to allow for a vTPM device, but the device will not be
provisioned for a guest unless the operator appropriately configures the
images and/or flavors.

Refer to the Nova documentation [2]_ for the specific extra specs and
properties that need to be set to provide vTPM devices to guests.

Barbican
--------

The swtpm library which provides emulated TPM devices encrypts secrets
locally in files on the file system. Nova uses the Barbican key manager
service for secret storage, which is already available as a charmed
application.

Conveniently, the default configuration for Nova will use the barbican
services from the keystone catalog to store the necessary secrets. These
secrets are scoped per project and the interactions with the secret store will
happen with appropriate context of the user. As such, there's no additional
information that the ``nova-compute`` charm requires in order to configure
the nova compute services so additional relations are unnecessary.

OpenStack Versions
------------------

This feature will be enabled for Wallaby and newer OpenStack releases.

Operating System Versions
-------------------------

This feature will be enabled for Ubuntu 20.04 (focal) and Ubuntu 22.04 (jammy).

Juju Version Dependencies
-------------------------

This feature has no dependency on Juju versions.

Alternatives
------------

There are no alternatives for vTPM support within the charms that integrates
nicely with OpenStack while using the OpenStack charms for deployment.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  billy-olsen

Gerrit Topic
------------

Use Gerrit topic "charm-vtpm" for all patches related to this spec.

.. code-block:: bash

    git-review -t charm-vtpm

Work Items
----------

- Add configuration changes to nova-compute charm
- Add functional tests to zaza-openstack-tests
- Provide user documentation around enabling the feature and how to use

Repositories
------------

No new repositories are required for this work.

Documentation
-------------

As part of this effort, the following documentation will need to be updated:

- Charm Deployment Guide
- Charm Readme
- Charm Guide
- Release Notes

Security
--------

The changes required in the charm do not introduce any security implications
above and beyond what is outlined in the Nova specification for enabling
emulated vTPM devices [1]_.

Testing
-------

Unit tests and functional tests will be implemented for this feature. The
functional tests will validate the various TPM device configurations and
validate that the TPM device is available within the guest.

Dependencies
============

* Nova Wallaby version or greater.

* swtpm TPM emulator [3]_ [4]_

* Focal-Wallaby support depends on swtpm package being backported to either
  the Wallaby Ubuntu Cloud Archive or Focal. Ubuntu developers have indicated
  a willingness to backport swtpm to Focal.



.. [1] https://specs.openstack.org/openstack/nova-specs/specs/victoria/implemented/add-emulated-virtual-tpm.html
.. [2] https://docs.openstack.org/nova/latest/admin/emulated-tpm.html
.. [3] https://bugs.launchpad.net/ubuntu/+source/swtpm/+bug/1948748
.. [4] https://github.com/stefanberger/swtpm
