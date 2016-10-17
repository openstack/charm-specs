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

===============================
Ceph Autotune
===============================

Ceph isn't as fast as it could be out of the box.

Problem Description
===================

Currently we don’t do any tuning of Ceph after installing it.  There’s
many little things that have to be tweaked to get ceph to be performant.

There are many mailing list messages of users becoming frustrated with
the slow performance of Ceph.  The goal of this epic is to capture the
low hanging fruit so everyone can have a better experience with Ceph.

Proposed Change
===============

I'm proposing several areas where easy improvements can be made.

There's several areas that can be improved such as HDD read ahead and
max hardware sector settings.  We can also tune the sysctl network
settings on 10 and 40Gb networks.

Finally I've noticed that IRQ alignment on multisocket motherboard
isn't ideal.  We can use numactl to pin the IRQs for all the components
in the IO path from network to osd to disk.

Of all the changes probably the IRQ pinning will be the most difficult.

Alternatives
------------

None

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  cholcombe973


Gerrit Topic
------------

Use Gerrit topic "performance-optimizations" for all patches related
to this spec.

.. code-block:: bash

    git-review -t performance-optimizations

Work Items
----------

1. HDD Read ahead settings: Often times the read ahead settings for
   hdd’s are too small for storage servers.

2. 10/40Gb sysctl settings: 10Gb and 40Gb networks need special sysctl
   settings to improve their performance.

3. HDD max hardware sectors: Making the max hardware sectors as large
   as possible allows bigger sequential writes and therefore better
   performance.

4. IRQ alignment for multisocket motherboards: It has been shown on
   multi socket motherboards that Linux does a bad job of lining up
   IRQ’s and results in lots of swapping of cache memory between CPU’s.
   This significantly degrades Ceph’s performance.

Repositories
------------

No new git repositories required for this work.

Documentation
-------------

Updates to the README's in the impacted charms will be made as part of this
change.

Security
--------

No additional security concerns.

Testing
-------

Code changes will be covered by unit tests; functional testing will be done
using a Mojo specification.

Dependencies
============

- charm-layering
