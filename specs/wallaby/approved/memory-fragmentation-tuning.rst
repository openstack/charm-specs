..
  Copyright 2021 Canonical Ltd.

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template.  If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

===========================
Memory Fragmentation Tuning
===========================

In some high memory pressure scenarios, the memory shortage would make the high
order pages hard to be allocated and also the page allocation would go to the
synchronous frequently reclaim thanks to the default gap between
min<->low<->high is too small to wake up the kswapd (asynchronous reclaim)
earlier.

Problem Description
===================

In the OpenStack compute node, especially the hyperconverged machine with the
Ceph OSDs using a lot of page cache. It is easy to have the memory allocation
stall issue. The issue would lead to several issues: The new instance cannot be
brought up (KVM needs to allocate order sixth pages) or VM stuck, etc. The
reasons are:

1). Compaction for big order page
If the THP (Transparent Huge Page) is used with the VM, it will be more severe
than the persistent huge pages reserved for the VM's dedicated usage. The THP
needs to allocate the 2MB (x86) huge pages at run time. Moreover, this is the
order 9 (2^9 * 4K = 2MB). In running system, it will be hard to get the
continuous 512 (2^9) 4K pages according to /proc/pagetypeinfo.

2). Synchronous reclaim.
There are three levels of watermark inside the system: 1). min 2). low 3).
high. When the number of free pages lowers down to the low watermark. The kswapd
will be wakened up to do the asynchronous reclaim. Furthermore, it will not be
stopped until the number of free pages reaches the high watermark. However, when
the memory allocation is strong enough, the free pages will continue to lower
down to the min watermark. At this point, the number of min pages is reserved
for emergency usage, and the allocation will go into the
direct-reclaim (synchronous) mode. This will stall the process.

Proposed Change
===============

In the past experience, the 1GB gap between min<->low<->high watermark is a good
practice in the server environment. The bigger gap can wake up the kswapd
earlier and avoid the synchronous reclaim. Moreover, this can alleviate the
latency. The sysctl parameters related to the watermark gap calculation:

vm.min_free_kbytes
vm.watermark_scale_factor

For the Ubuntu kernel before 4.15 (Bionic), the only way to tune the watermark is
to modify the vm.min_free_kbytes. The gap would be 1/4 of the
vm.min_free_kbytes. However, increasing the min_free_kbytes is the minimum
watermark reservation increase, which will decrease the actual memory that the
runtime system can use.

For Ubuntu kernel after 4.15, vm.watermark_scale_factor can be used to increase
the gap without increasing the min watermark reservation. The gap is calculated
by "watermark_scale_factor/10000 * managed_pages".

The proposed solution is to set the 1GB watermark gap by using the above two
parameters when the compute node is rebooted.

The feature will be designed in flexible ways:
1). There will be a switch to turn on/off the feature. By default, it is turned
off. For some small memory compute nodes (<32GB), the 1GB low memory is too
many.

2). The manual config has a higher priority to overwrite the default calculation.

Alternatives
------------

The config can be set up in the run time with the following command:
juju deploy cs:sysconfig-2
juju add-relation sysconfig nova-compute
juju config sysconfig sysctl="{vm.extfrag_threshold: 200,
vm.watermark_scale_factor: 50}"

However, each system might have different memory capacities. The
watermark_scale_factor needs to be calculated manually.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
- Gavin Guo <gavin.guo@canonical.com>


Gerrit Topic
------------

Use Gerrit topic "memory-fragmentation-tuning" for all patches related to this spec.

.. code-block:: bash

    git-review -t memory-fragmentation-tuning

Work Items
----------

Implement the watermark_scale_factor value calculation to set up the gap to 1GB.

Repositories
------------

No new git Repository is required.

Documentation
-------------

The documentation is needed to include the switch to turn on/off the feature.

Security
--------

The use of this feature exposes no other security attack surface.

Testing
-------

To verify if the calculated watermark value is correct. Also, in different
kernel versions, different parameters should be used (min_free_kbytes v.s.
watermark_scale_factor).

Dependencies
============
None
