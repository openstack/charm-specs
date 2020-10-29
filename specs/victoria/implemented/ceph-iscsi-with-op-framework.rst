..
  Copyright 2020 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

==========================================
Ceph iSCSI Gateway with Operator Framework
==========================================

The aim of this piece of work is to produce an Ceph iSCSI Gateway charm
written in the operator framework.

Problem Description
-------------------

There is currently no charmed mechanism for providing Ceph backed block devices
to clients which cannot mount rbd devices directly. In addition a new charm
framework is under development and this charm is an opportunity to test it for
writing OpenStack charms.

Proposed Change
---------------

To write a ceph-iscsi charm using the operator framework. The charm will
be marked as a preview charm but should have functional equivalence with a
corresponding OpenStack charm written using the reactive framework.

The ceph-iscsi charm should interact with the Juju environment via the
operator framework. This means that calls to libraries like
charmhelpers.core.hookenv should be avoided. This applies to all aspects
of the charm including interfaces.

What versions of the operating system are affected or required?
Focal or Bionic + HWE Kernel

What version of Juju is required?
2.7+

Alternatives
~~~~~~~~~~~~

Write the charm using classic or reactive framework.

Implementation
--------------

Assignee(s)
~~~~~~~~~~~
Primary assignee:
  Liam Young <lp:gnuoy>

Gerrit Topic
~~~~~~~~~~~~

Use Gerrit topic "<topic_name>" for all patches related to this spec.

.. code-block:: bash

    git-review -t ceph-iscsi

Work Items
~~~~~~~~~~

The implementation consists of the following items:

- Validate that Ubuntu ceph-iscsi packages are working on Focal
- Plan charm interfaces, supporting modules etc
- Decide how dependencies are to be handled (pip, git submodulees etc)
- Decide on how framework and dependency versions will be
  pinned during development and upon release.
- Write basic charm to deploy and configure gateway services.
- Write functional tests to perform a multipath mount a iscsi volume.
- Write README
- Certificates interface
- Security checklist
- Add pause  & resume
- Implement ceph-iscsi api security (admin_password, trusted_ips etc)
- zaza tests for creating nd mounting a target, doing basic i/o and
  initiator reconnect testing.
- Add iscsi target create action
- Ensure update statue detects missing or incomplete relations.
- Add series upgrade

Repositories
~~~~~~~~~~~~

This will almost certainly change but:

- https://github.com/openstack-charmers/ceph-iscsi
- https://github.com/openstack-charmers/ops-openstack
- https://github.com/openstack-charmers/oper-interface-ceph-client

Documentation
~~~~~~~~~~~~~

- Any additional documentation will be considered for `charm-deployment-guide`.

Security
~~~~~~~~

The gateway api will be secured with a password and access limited to the IP
addresses of the other gateways.

The certificates interface will also be implemented to encrypt the api
endpoint.

Testing
~~~~~~~

The charm will have the standard lint, unit and functional tests. The
functional tests will be written in zaza.

Dependencies
------------

- The operator framework having sufficient maturity.
- A way to reuse existing code is identified. It would probably be unacceptable
  to add a third version of common functions.
