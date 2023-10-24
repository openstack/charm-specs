..
  Copyright 2023 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..

===================================
Unify ceph nova-compute credentials
===================================

Ceph credentials should be the same across all nova-compute charm apps
to allow live-migration of VMs booted from image (and stored in ceph
via use of libvirt-image-backend=rbd config) to succeed.


Problem Description
===================

Currently every nova-compute charm app registers a ceph credential named
after the charm app itself. Therefore, if a cloud has two nova-compute charm
apps named nova-compute-haswell and nova-compute-skylake, then their ceph
credentials will be the respective names.

The result of this is that live-migrating a VM booted from image where both
nova-compute charm apps have been configured to use libvirt-image-backend=rbd
(resulting in the image being stored in ceph) fails because the libvirt XML
for the instance will have the ceph credential and key associated with the
name on the source, to which the destination does not possess the key to access
the image, resulting in a failed migration due to permission denied error, as
documented in [1]_.


Proposed Change
===============

In order to solve the problem, all nova-compute charm apps need to have access
to the same ceph credentials and by extent, to the same key, allowing
credentials specified in the libvirt XML to be compatible with both the host
and destination.

To achieve that, it is necessary to move away from having ceph credentials
deriving their name from the nova-compute charm app names. Here we propose
a new name to be a common one between all of
them: 'nova-compute-ceph-auth-<secret-uuid-first-block>'.

There are many reasons for choosing the above-mentioned new credentials name:

1) As part of the name transition, it is necessary to make changes to the
   libvirt secret entry. The secret entry in libvirt consists of:

   secret UUID - name/usage - key

   For each registered libvirt secret, we can only update the key. In order
   to allow continued function of existing VMs as we upgrade the charms
   performing the credential transition, we cannot delete and replace the
   entire secret. It is necessary then to add a new secret with the new
   credentials and new UUID, while also preserving the old secret.

2) The upgrade path from the old name to the new name is simpler if every
   single existing name is handled as old. For example, if we chose
   'nova-compute' to be the new name, then deployments that had nova-compute
   charm apps named 'nova-compute' would need to be handled differently. The
   main technical difficulty of this approach is again, the libvirt secret
   management limitations, preventing us from adding a new libvirt secret
   with the same name and a new UUID, nor can we achieve the desired
   functionality while avoiding the need of adding a new secret UUID.

3) Having a credential name that is very unlikely to clash with any
   charm app name avoids the situation described above. By using the first
   block of the secret UUID in the credential name we also make it easier to
   identify the credential as being the new one. If more credential names are
   required to be performed in the future, by following this pattern we can
   easily have versioning between them.


Charm Impact
------------

Only the nova-compute charm is affected by the code changes. The ceph-mon
charm does not require code changes.


Upgrade Impact
--------------

On upgrade, transitioning from old credentials to new credentials consists of:

1) Every nova-compute unit needs to send the new application-name to
   ceph-mon. Ceph-mon will then register the new credential when receiving
   it for the first time (subsequent receipt of the new application-name
   will not need to re-register the new credentials, as they would already
   exist) and send back the key of the new credentials.

2) Nova-compute units, upon receiving the updated key, will rewrite the
   config files with the new credentials, keys and UUID, and invoke libvirt
   to register the new secret. New VMs created from this point onwards will
   use the new credentials.

The nova-compute units will retain their old credentials key file in their
/etc/ceph folder and registered in libvirt, so existing workloads are not
affected and allows the units to be able to receive live-migration of
non-rebooted instances.


Existing Workloads Impact
-------------------------

Existing running VMs will have their old credentials declared in their
libvirt XML and continue to run fine as the old credentials as retained.
A full power cycle of those VMs is required to update their libvirt XML
and switch to the new credentials. Those VMs cannot be migrated to
nova-compute nodes operated by different nova-compute charm apps until
rebooted.


Scale-out Impact
----------------

New nodes deployed with the updated charm need to install both old and new
credentials. To achieve this, the nova-compute code will need to send the old
application-name on ceph-relation-joined, then upon successful configuration
of the old credential, send the new application-name and trigger the upgrade
path. The end result is that a newly deployed node will have both old and new
credentials and can receive migration of non-rebooted VMs that are still using
the old credentials. This was a concern already pointed out by [2]_.


Revert/Downgrade Possibility
----------------------------

Unfortunately, downgrading and reverting the changes in case something
unexpected happens is not smooth, but is fairly straightforward. The
main problem is that the current code does not have a way to re-send
the application-name to ceph-mon outside of the ceph-relation-joined hook
function, so this information has to be exchanged manually through the
juju relation-set command. This is enough to revert all the changes.

If something went wrong during the upgrade or downgrade, then libvirt secrets
may need to be manually maintained to address a potential error or conflict of
names, UUID, or re-set the keys.


Charm Configuration Options
---------------------------

No charm config options are affected or have to be added for this change.
The change, however, only affects users that have the config
libvir-image-backend set to 'rbd'.


Configuration Files
-------------------

The config files are affected in the following way:

* */etc/ceph/secret.xml* - This file will be updated to have the new UUID
  and credential name. It is only used for the secret to be registered in
  libvirt once.

* */etc/ceph/ceph.client.nova-compute-ceph-auth-<secret-uuid-first-block>* -
  This file will be added with the key for the new credentials.

* */etc/nova/nova.conf* - The properties rbd_user and rbd_secret_uuid will be
  updated with the new credentials and UUID respectively.


Non-Charm Configuration
-----------------------

The relation-data changes between the nova-compute and ceph charms affect
the following 2 fields:

* *application_name* - Ceph-mon will receive the application name
  nova-compute-ceph-auth-<secret-uuid-first-block> instead of the
  nova-compute charm app name.

* *key* - Nova-compute will receive the new credential's key instead of the
  old credential's key.


OpenStack Versions
------------------

This feature will be enabled for Yoga and newer OpenStack releases.

Operating System Versions
-------------------------

This feature will be enabled for Ubuntu 20.04 (focal) and newer Ubuntu
releases.

Juju Version Dependencies
-------------------------

This feature has no dependency on Juju versions.

Alternatives
------------

A design alternative is possible to achieve the same result, although
with some advantages and disadvantages:

* Change the relation-data to exchange a dictionary containing all
  nova-compute charm app names and keys between all nova-compute units. To
  achieve this, either the nova-cloud-controller needs to be involved (as it
  already currently is for exchanging SSH keys), or a new relation needs to be
  created between nova-compute between different charm apps. This alternative
  requires more code changes and relation data structure changes, but the end
  result is generally more consistent and resilient against unexpected
  behavior, such as hook errors or hooks running out of order.


Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ganso

Gerrit Topic
------------

Use Gerrit topic "lp2028559" for all patches related to this spec.

.. code-block:: bash

    git-review -t lp2028559


Repositories
------------

No new repositories are required for this work.

Documentation
-------------

As part of this effort, the following documentation will need to be updated:

- Charm Guide
- Release Notes

Security
--------

The changes required in the charm do not introduce any additional security
implications beyond the security requirements and compromises already in place.

The extra credentials in ceph and extra keys in nova-compute are subject to the
same vulnerability as the existing credentials and keys.

Testing
-------

Unit tests and functional tests will be implemented for this feature. The
functional tests will validate whether the nova-compute nodes have
credential files for both the name derived from their charm app name, and
the new proposed unique name.

Currently there are not multiple nova-compute charm apps configured in
the CI, nor their presence supported by the functional tests code, nor any
live-migration tests. As future work there could be live-migration tests
across different nova-compute charm apps.

Work Items
----------

- Implement code changes in nova-compute charm
- Add functional tests to zaza-openstack-tests
- Provide user documentation on impact of changes


Dependencies
============

No hardware, software or version dependencies are required for this change
to be functional.


References
==========

.. [1] https://launchpad.net/bugs/2028559
.. [2] https://launchpad.net/bugs/2037003

