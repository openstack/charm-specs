..
  Copyright 2016, Canonical UK

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

====================
Swift Global Cluster
====================

Add support for Swift Global Cluster feature to Swift charms.

Problem Description
===================

At the moment Swift charms do not support the Swift Global Cluster feature
which is already available in the upstream:

https://docs.openstack.org/swift/latest/overview_global_cluster.html

As a result it is not possible to deploy Swift clusters in a multi-region mode
using Swift charms.

Proposed Change
===============

The proposed change can be split to the following items:

#. Add new config options to swift-storage charm to handle the following
   parameters:

- "region":

  purpose: specifies region to request membership
  data type: int
  valid values: natural number

- "account-server-port-rep":

  purpose: specifies listening port of the swift-account-replicator service
  data type: int
  valid values: 0 - 65535

- "container-server-port-rep":

  purpose: specifies listening port of the swift-container-replicator service
  data type: int
  valid values: 0 - 65535

- "object-server-port-rep":

  purpose: specifies listening port of the swift-object-replicator service
  data type: int
  valid values: 0 - 65535

#. Add new config options to swift-proxy charm to handle the following
   parameters:

- "enable-multi-region":

  purpose: specifies whether the Swift Global Cluster feature should be used
  data type: boolean
  valid values: True / False

- "read-affinity":

  purpose: specifies which backend servers to prefer on reads
  data type: string
  valid values: matches "^r\d+z?(\d+)?=\d+(,\s?r\d+z?(\d+)?=\d+)*$" pattern

- "write-affinity":

  purpose: specifies which backend servers to prefer on writes
  data type: string
  valid values: matches "^r\d+(,\s?r\d+)*$" pattern

- "write-affinity-node-count":

  purpose: specifies how many local objects will be tried before falling back
  to non-local ones
  data type: string
  valid values: matches "^\d+(\s\*\sreplicas)?$" pattern

#. Update template files to contain options required by the Swift Global
   Cluster feature.

#. Rename and change location of template configuration files responsible for
   Swift account, container and object services to match the schema required by
   the Swift Global Cluster feature.

#. Extend relation data exchanged between swift-storage and swift-proxy nodes
   with the region and other replication information.

#. Implement 'rings-distributor' - 'rings-consumer' relation based on a common
   interface ('swift-global-cluster') in swift-proxy charm to handle rings
   distribution across all proxy nodes participating in the multi-region
   deployment.

#. Update test files.

#. Update the documentation with information on how to setup Swift in
   multi-region mode.

Alternatives
------------

At the moment there is no alternative to the Swift Global Cluster feature.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  tkurek

Gerrit Topic
------------

Use Gerrit topic "swift-global-cluster" for all patches related to this spec.

.. code-block:: bash

    git-review -t bug/1815879

Work Items
----------

Update swift-proxy charm
++++++++++++++++++++++++

- Update 'config.yaml' file to add new options
- Update template files with new options
- Update 'metadata.yaml' file to add the new relation
- Add hook files as symlinks to the 'swift_hooks.py' file
- Update 'swift_hooks.py' file with hook definitions for the new relation
- Update test files
- Update 'README.md' file to describe the feature
- Update other files (if needed) to implement the feature

Update swift-storage charm
++++++++++++++++++++++++++

- Update 'config.yaml' file to add new options
- Rename and change location of template configuration files responsible for
  Swift account, container and object services to match the schema required by
  the feature.
- Update 'metadata.yaml' file to add extra bindings ('cluster' and
  'replication')
- Update 'swift_storage_relation_joined' hook definition with extended relation
  data
- Update test files
- Update 'README.md' file to describe the feature
- Update other files (if needed) to implement the feature

Repositories
------------

No new git repositories will be required to implement this feature.

Documentation
-------------

The README files of swift-proxy and swift-storage charms should be updated with
instructions on how to setup Swift in multi-region mode.

Security
--------

No security implications for this change.

Testing
-------

Unit and functional tests of swift-proxy and swift-storage charms will need to
be updated to support implementation of this feature.

Dependencies
============

No external dependencies.
