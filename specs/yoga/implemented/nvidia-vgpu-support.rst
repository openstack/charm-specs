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

==================================
NVidia Virtual GPU support in Nova
==================================

Graphical Processing Units (GPUs) are optimized for performing various
mathematical operations in silicon. These optimizations allow for extremely
fast floating point operations that benefit various workloads such as graphics
rendering, machine learning and others. Hardware manufacturers such as
Nvidia [0] and Intel [1] offer physical GPU (pGPU) cards that can be virtually
partitioned into virtual Graphics Processing Units (vGPUs), enabling multiple
guests to make use of a single physical GPU installed in the system.

Recent changes in Nova [2][3][4][5] introduced partial support for exposing
physical GPUs as virtual GPUs to Nova instances. This spec is about enabling
this support for NVidia hardware in a charm deployment.

Problem Description
===================

Machine Learning workloads benefit heavily from the hardware capabilities of
GPUs. The common configuration for running ML based workloads leverage
Kubernetes, which has poor multi-tenancy support and is often run on top of
OpenStack to provide elastic resources and provide multi-tenant isolation.

Currently, deployments of Charmed OpenStack must use GPU passthrough [6] in
order to enable a guest to make use of a GPU. This limits deployments to
require one physical card for each guest that they intend to run on the
hypervisor. vGPU support enables multiple guests to make use of the installed
pGPUs which enables deployments to increase the number of guests which can
leverage the system.

Proposed Change
===============

The Nova project within OpenStack has provided vGPU support since the Queens
release. [2][3] The Nova project continues to iterate over the design of vGPU
support, making enhancements to allow multiple vGPU types in the Ussuri
release. [4][5]

The Nova project provides support for vGPUs for allocation and assignment. The
vGPU support upstream has evolved and thus, there are different capabilities
available in different releases of OpenStack. The primary difference is whether
or not multiple GPU devices are supported and how to specify which devices
should be used for virtual GPU types.

A new nova-compute-nvidia-vgpu charm, subordinate to nova-compute, will be
created. It will install the NVidia host software and set it up if and only if
the presence of one or several such pGPU cards is detected. Indeed deploying
the subordinate charm "blindly" to all compute hosts, no matter which of them
provide NVidia pGPU cards or not, simplifies Day-0 and Day-1 operations. If no
supported pGPU card is found, the subordinate charm will simply state so in
its unit status message.

The proprietary software package (.deb) [7] to be installed on host will be
passed to the subordinate charm as a charm resource.

To configure vGPUs, a new config option ``vgpu-device-mappings`` will be added
to the nova-compute-nvidia-vgpu charm. The format of this configuration value
will be a YAML-formatted dict (or JSON-formatted, since JSON is valid YAML)
where the key is the vgpu_type and the value is a list of device addresses.
E.g.

.. code-block:: none

    juju config nova-compute-nvidia-vgpu vgpu-device-mappings="$(cat << EOF
    vgpu_type1:
      - device_address1
      - device_address2
    vgpu_type2:
      - device_address3
    EOF
    )"

is equivalent to

.. code-block:: none

    juju config nova-compute-nvidia-vgpu vgpu-device-mappings=\
        "{'vgpu_type1': ['device_address1', 'device_address2'], 'vgpu_type2': ['device_address3']}"

vGPU support will be added to the following Operating System versions:

- Bionic
- Focal

vGPU support will be enabled for the following OpenStack releases:

- Queens (bionic only)
- Rocky (migration path only)
- Stein (migration path only)
- Train (migration path only)
- Ussuri
- Victoria
- Wallaby
- Xena
- Yoga (and newer)

Juju 2.9 or newer is required.

Alternatives
------------

Using PCI passthrough [6] virtual machines can have direct access of the
physical GPUs installed in the compute host. This limits however deployment to
require one physical card for each guest.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

Aurelien Lourot <lp:aurelien-lourot>

Gerrit Topic
------------

Use Gerrit topic "vgpu-support" for all patches related to this spec.

.. code-block:: none

    git-review -t vgpu-support

Work Items
----------

The implementation consists of the following items:

- Create a nova-compute-nvidia-vgpu subordinate charm with a
  ``vgpu-device-mappings`` config option.

- Let the subordinate charm feed the principal charm with:

  * vGPU-related Nova configuration bits, using a ``subordinate_configuration``
    key in order to pass a JSON blob [8][9] over a new ``nova-vgpu`` relation.
    The JSON blob will be a 1:1 mapping of the wanted Nova
    configuration, [3][5] essentially containing the enabled vGPU types and the
    device addresses for each vGPU type.
  * Data around installed packages and services in order to properly implement
    pausing, resuming and upgrading (examples of this mechanism can be found
    here [10][11]).

- Let the subordinate charm install the proprietary host software [7] whenever
  the required hardware is found:

  * in the install hook,
  * after a reboot, either via a hook or an action.

- Let the subordinate charm perform any necessary action for setting up the
  proprietary software. [7]

- See "Documentation" and "Testing" sections below.

Repositories
------------

- ``charm-nova-compute``
- ``charm-nova-compute-nvidia-vgpu`` (new)
- ``charm-guide``
- ``charm-deployment-guide``

Documentation
-------------

Documentation will be written for ``charm-guide`` and/or
``charm-deployment-guide`` explaining the usage of this feature, i.e. how to
use and configure the charms. Useful links around these topics will be
provided:

- guest flavors,
- proprietary guest software, [7]
- license server.

Security
--------

Special care will be taken to prevent injecting arbritrary Nova configuration
through the ``vgpu-device-mappings`` config option.

The proprietary host software [7] will only be installed if suitable hardware
is found.

Testing
-------

Gate tests (unit and functional tests) will be added in order to cover the
new charm's behaviour without any special hardware.

If needed and possible, functional tests exercising special GPU hardware and
NVidia proprietary software [7] will be written and run periodically.

Dependencies
============

We depend on proprietary NVidia software [7] to be provided to the subordinate
charm as a .deb resource file.

----------

| [0]: https://docs.nvidia.com/grid/5.0/pdf/grid-vgpu-user-guide.pdf
| [1]: https://01.org/igvt-g
| [2]: https://specs.openstack.org/openstack/nova-specs/specs/queens/implemented/add-support-for-vgpu.html
| [3]: https://docs.openstack.org/nova/queens/admin/virtual-gpu.html
| [4]: https://specs.openstack.org/openstack/nova-specs/specs/ussuri/implemented/vgpu-multiple-types.html
| [5]: https://docs.openstack.org/nova/ussuri/admin/virtual-gpu.html
| [6]: https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/app-pci-passthrough-gpu.html
| [7]: https://docs.nvidia.com/grid/index.html
| [8]: https://opendev.org/openstack/charm-nova-compute/src/branch/master/hooks/charmhelpers/contrib/openstack/context.py#L1508
| [9]: https://opendev.org/openstack/charm-ceilometer-agent/src/branch/master/hooks/ceilometer_hooks.py#L85
| [10]: https://opendev.org/openstack/charm-nova-compute/src/branch/master/hooks/nova_compute_utils.py#L76
| [11]: https://opendev.org/openstack/charm-ceilometer-agent/src/branch/master/hooks/ceilometer_hooks.py#L86
