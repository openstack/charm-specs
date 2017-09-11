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
Ceph Charms Layering
====================

The Ceph charms need to shed technical debt to allow faster iteration.

Problem Description
===================

There is lots of duplicate code in the ceph, ceph-mon, ceph-osd and
ceph-radosgw charms.  The aim is to eliminate most of that using the
new Juju layers approach.

Proposed Change
===============

The new layering/reactive way of charming offers the Ceph charms significant
benefits.  Between ceph-mon, ceph-osd, ceph and ceph-radosgw we have a lot of
duplicate code.  Some of this code has been pulled out and put into the
charmhelpers library but that can't be done for everything.  The juju layering
approach to charms will allow the rest of the shared code to be consolidated
down into a base layer.

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  ChrisMacNaughton


Gerrit Topic
------------

Use Gerrit topic "layering" for all patches related to this spec.

.. code-block:: bash

    git-review -t layering

Work Items
----------

ceph base layer
+++++++++++++++

These is the biggest piece of the project.  All duplicate code that is
shared between ceph ceph-mon and ceph-osd will be consolidated down
into this base layer.

ceph-mon layer
++++++++++++++

Create a ceph-mon layer that utilizes the ceph base layer.  Once the base layer
is settled the next step will be to build the ceph-mon charm on top of the base
layer.

ceph-osd layer
++++++++++++++

Create a ceph-osd layer that utilizes the ceph base layer.

layer ceph charm
++++++++++++++++

Recreate the ceph charm by combining the ceph-osd + ceph-mon layers.  With
layering the ceph charm essentially becomes just a combination of the other
layers.  We get this almost for free and it should be easy to maintain all
3 charms going forward.

Repositories
------------

The following repositories will need to be migrated to the /openstack
namespace with appropriate unit testing and CI:

1. https://github.com/ChrisMacNaughton/charm-interface-ceph-client
2. https://github.com/ChrisMacNaughton/charm-interface-ceph
3. https://github.com/ChrisMacNaughton/charm-interface-ceph-radosgw
4. https://github.com/ChrisMacNaughton/charm-interface-ceph-osd
5. https://github.com/ChrisMacNaughton/charm-layer-ceph-mon
6. https://github.com/ChrisMacNaughton/charm-layer-ceph-osd

Documentation
-------------

There shouldn't be any documentation changes needed for this.  The aim to
have identical functionality to what was there before.

This will impact developer workflow because at some point we're going to
have to stop all posts to all the repos and cut over to this new version
of the charm. The repositories are also going to change. The goal is to
publish them layered version of the charm rather than the compiled bits.

The test automation will then compile the layered charm and use that for
testing instead of just cloning the repo like it does today.

Security
--------

No additional security concerns.

Testing
-------

Because this is such a radical change from what we had before the only
way to accurately test it is to show identical functionality to what we
had before. A mojo spec will be used to demonstrate this.

Dependencies
============
None
