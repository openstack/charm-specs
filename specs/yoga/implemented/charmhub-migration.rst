..
  Copyright 2021, Canonical UK

  This work is licensed under a Creative Commons Attribution 3.0
  Unported License.
  http://creativecommons.org/licenses/by/3.0/legalcode

..
  This template should be in ReSTructured text. Please do not delete
  any of the sections in this template. If you have nothing to say
  for a whole section, just write: "None". For help with syntax, see
  http://sphinx-doc.org/rest.html To test out your formatting, see
  http://www.tele3.cz/jbar/rest/rest.html

================================
Charmstore to Charmhub Migration
================================

This spec proposes a new, better, but alternative means for maintaining and
releasing OpenStack charms in a world with a new charm store (charmhub.io)
which provides richer mechanisms for charm delivery.

A new charm store, charmhub, has been introduced that shares features and
capabilities with snaps. The legacy charm store, jaas.ai, is planned to be
retired and all projects delivering charms should migrate to charmhub. One of
the compelling features of charmhub is the ability to offer multiple tracks for
releases.

The charms themselves are written to support multiple releases of OpenStack
from pre-Mitaka to Xena. This is convenient and straightforward for having
development working only on one branch at a time for a charm, but requires that
code and dependencies work towards the lowest common supported denominator.
This is problematic as these charms depend on python libraries that are moving
forward and dropping support for legacy python versions such as Python 2, 3.5
and beyond, which causes maintenance challenges for current charms.

Problem Description
===================

Currently, OpenStack Charms are delivered in two charmstore namespaces -
``openstack-charmers`` and ``openstack-charmers-next``. The use of two
namespaces predates the existence of channels within the jaas.ai charm store.
These namespaces are equivalent to the stable and edge channels respectively.

The git repositories hosting the source code for the OpenStack charms generally
live on opendev.org git servers. The repositories contain branches of each
stable charm release and an unstable development branch (master). A post-merge
job in the CI system publishes an updated version of the charm to the jaas.ai
charm store after any change is merged. Changes landing in the master
development branch are published in the openstack-charmers-next namespace and
changes landing in the most recent stable branch are published in the
openstack-charmers namespace.

One of the challenges with the current approach is that the two branches must
focus on the lowest common denominator for dependencies; meaning that most
python libraries leveraged and included in the charms must be limited to
versions that support older python releases that are still included on older
OpenStack releases. As the world of Python library development moves on,
support for older python versions are being deprecated and removed causing a
variety of maintenance headaches for the current charms.

Another challenge with this approach is that each new feature introduced adds
potential risk to older install bases. In order to mitigate this, the gate
testing of the OpenStack Charms runs a broad sampling of tests across a
multitude of OpenStack and base distro releases. New contributors are often
running into challenges with the CI system and needing to debug older releases
even when their patches are not relevant to these older versions.

One of the positive sides to this arrangement is that development primarily
only needs to consider 2 branches at a time - the master development branch
and the most recent stable branch. All patches are primarily targeted at the
master development branch and relevant bug fixes are backported to a single
release.

Proposed Change
===============

OpenStack charms will adopt a model of release and delivery which takes
advantage of the various delivery channels offered by charmhub which closely
matches channels in the `Snap store`_. Note, that not all charms will support
all releases/versions of a specific service and charms will only adopt branches
and tracks for those releases which are relevant to the service itself.

Tracks
------

Tracks will be set up to track git branches in the upstream repositories and
will result in a track per release of the charm payload (e.g. the software
service it delivers). When the primary software payload of the charm uses
well-known release names, the release name will be used in lieu of version
numbers for the track name. When the primary software payload of the charm does
not use well-known release names, the version of the software will be used.

For example, the nova-compute charm delivers nova hypervisor services and the
charm tracks will correspond to the OpenStack release of the nova services
installed (e.g. ussuri, victoria, etc.).

As another example, the OVN Central charm delivers OVN as the payload and OVN
uses release names/versions of YY.MM. In this scenario, the OVN charm Central
charm will provide a track corresponding to the YY.MM naming schema (e.g.
21.03, 21.09, etc).

Risk Levels
-----------

The charmhub store also has the ability to allow the developers and users to
provide and consume different risk levels. This feature was also available in
the legacy charmstore, however it was not leveraged by the release process
until very recently. There are 4 risk levels available within any track: edge,
beta, candidate, and stable. The CI system will automatically publish a new
charm for any change which lands in the corresponding git branch to the edge
risk level.

Charms will be promoted through the edge -> beta -> candidate -> stable risk
levels by a human actor, not through an automated script. Tooling may be
developed to assist in this process, but automatic promotion is out of scope at
this time. Future enhancements may include triggering a promotion of a build
upon the introduction of git tags, but that is considered out of scope for this
work.

Branches
--------

The charmhub store also provides for a further subdivision of tracks and risk
levels into branches, which are meant to provide temporary, short-lived,
releases for bug fixing purposes. There are various possibilities to consider
regarding how branch flows should be worked into the development flow of
OpenStack charms.

For the purposes of this work, branches are explicitly out of scope and should
be considered separately.

Git Branches
------------

Git branches will need to be created for the various tracks that are delivered
in Charmhub. The relationship between git branches and charmhub tracks are
one-to-many, meaning that a change landing in one branch may result in a charm
being published into multiple tracks. This is useful when the tracks provide
logically different targets, while the branch serves the same functionality.
For example, the latest master branch of development will logically point to
the latest OpenStack release as well as the latest/edge release.

Additionally, the charms in each of the git branches and tracks are intended to
support the targeted release N and the previous release N-1 for upgrade
purposes.

OpenStack Charms
----------------

For those charms specifically delivering OpenStack services a new git branch
will be created for each OpenStack release the charm supports. This branch will
be named in the form of ``stable/<openstack-release>`` for all stable versions
of OpenStack. Branches will only be created back to the Queens release of
OpenStack. Each of these branches will be created from the tip of the current
stable charms release (stable/21.10) as a base. These branches will differ from
each other regarding the primary version of software that they are targeting as
well as the default configuration values for the track they are deployed from.

+-----------------+-------------------+
| Git Branch      | Charmhub Track(s) |
+=================+===================+
| stable/queens   | train             |
| stable/rocky    |                   |
| stable/stein    |                   |
| stable/train    |                   |
+-----------------+-------------------+
| stable/ussuri   | ussuri            |
+-----------------+-------------------+
| stable/victoria | victoria          |
+-----------------+-------------------+
| stable/wallaby  | wallaby           |
+-----------------+-------------------+
| stable/xena     | xena              |
+-----------------+-------------------+
| stable/yoga     | yoga              |
+-----------------+-------------------+
| master          | zed, latest/edge  |
+-----------------+-------------------+

Ceph Charms
-----------

Currently, Ceph updates are delivered through the Ubuntu Cloud Archive which
ties the ceph charms to OpenStack versions, causing some challenges in defining
tracks per Ceph release. In scenarios where services are co-located (e.g.
nova-compute and ceph-osd), updating the source repository for the ceph-osd
charm will have unintended consequences on the nova-compute services and their
versions. Rather than introduce complex pinning or a new package archive, the
charms used to deliver Ceph will track the OpenStack release names in addition
to the Ceph release names, up to the Octopus release. This is expected to make
the migration easier for users.

The development git branches will be created corresponding to the appropriate
Ceph version where the product was delivered. In order to make it easier for
users to migrate, Ceph charms may have a single branch delivered to multiple
Charmhub tracks.

+-----------------+-------------------------+
| Git Branch      | Charmhub Track(s)       |
+=================+=========================+
| stable/luminous | luminous, queens, rocky |
+-----------------+-------------------------+
| stable/mimic    | mimic, stein            |
+-----------------+-------------------------+
| stable/nautilus | nautilus, train         |
+-----------------+-------------------------+
| stable/octopus  | octopus                 |
+-----------------+-------------------------+
| stable/pacific  | pacific                 |
+-----------------+-------------------------+
| master          | quincy, latest/edge     |
+-----------------+-------------------------+

On the newer branches, a mapping can be made to safely convert a Ceph release
name into a cloud-archive pocket in a way that doesn’t unintentionally upgrade
either one, with the assumption that a valid mapping between OpenStack and Ceph
versions is clearly documented. What that mapping should look like:

+-------------------+---------------------------------------------+
| Ceph Release      | OpenStack Release                           |
+===================+=============================================+
| Luminous          | Queens                                      |
+-------------------+---------------------------------------------+
| Mimic             | Stein                                       |
+-------------------+---------------------------------------------+
| Nautilus          | Train                                       |
+-------------------+---------------------------------------------+
| Octopus           | Ussuri                                      |
+-------------------+---------------------------------------------+
| Pacific           | Wallaby (mid-cycle Openstack, more support) |
+-------------------+---------------------------------------------+
| Unstable / Quincy | Yoga                                        |
+-------------------+---------------------------------------------+

The above works because we can include distro (bionic-queens) with Rocky
safely, and we can include Pacific’s focal-wallaby with Xena safely, as the
Xena packages have higher versions.

The Ceph charms include the following charms:

* ceph-fs
* ceph-iscsi
* ceph-mon
* ceph-osd
* ceph-proxy
* ceph-radosgw
* ceph-rbd-mirror
* ceph-dashboard

Provider-specific Subordinate Charms
------------------------------------

Some services such as Cinder and Neutron allow for provider specific
subordinate charms to be used in order to allow for a specific SDN, storage
plugin, etc. These provider-specific subordinate charms are considered
supporting charms rather than OpenStack specific charms, however they are often
enabling specific functionality for a specific storage backend. As such, they
will follow the OpenStack Charms track schema.

These charms will include the following subordinate charms, and as of this
writing include:

* cinder-ceph
* cinder-lvm
* cinder-netapp
* cinder-purestorage
* neutron-openvswitch
* neutron-api-plugin-arista
* neutron-api-plugin-ovn
* keystone-saml-mellon

OVN Charms
----------

OVN is a networking SDN which is used as the primary SDN for Charmed OpenStack
deployments. OVN has been provided through the Ubuntu Cloud Archive as well as
the Ubuntu archives. The OVN charms are a bit of a mix when it comes to
providing source versions. Standalone charms such as OVN Dedicated Chassis and
OVN Central have source options which allow for configuring the archive to use
for package installation. As a subordinate charm, OVN Chassis will adopt the
version that is installed alongside the principal charm.

As OVN is a general SDN and not implemented specifically for OpenStack, the
tracks in charmhub should closely follow the versions of the software delivered
rather than the OpenStack release. This may be a bit confusing at first for
users migrating, but this should be addressed with clear documentation as well
as tooling to help the end-users select the appropriate track to upgrade their
charms to. Additionally, tracks will be put in place labelled
openstack-{release-name} to ease the migration burden.

OVN charms will have the following sets of branches and tracks:

+--------------+----------------------------+
| Branch       | Track(s)                   |
+==============+============================+
| stable/20.03 | 20.03, ussuri, victoria    |
+--------------+----------------------------+
| stable/20.12 | 20.12, wallaby             |
+--------------+----------------------------+
| stable/21.09 | 21.09, xena                |
+--------------+----------------------------+
| stable/22.03 | 22.03                      |
+--------------+----------------------------+
| master       | latest/edge                |
+--------------+----------------------------+

Supporting Charms
-----------------

In the OpenStack ecosystem, there are other core services which are required to
produce an OpenStack cloud. These services include messaging, database, and
certificate management services. While they are critical to the functionality
of an OpenStack cloud, they are not tied to specific releases of OpenStack and
these charms should result in tracks which are appropriate for their specific
behavior.

These charms consist of the following services, and will result in tracks that
are based on the versions of software provided. These services are generally
delivered from the Ubuntu archives (not the Ubuntu Cloud Archive) and will
track the versions of their versions in Ubuntu.

+----------------------+---------------+-------------------------+
| Charm                | Branch        | Track(s)                |
+======================+===============+=========================+
| hacluster            | stable/bionic | 1.1.x                   |
+----------------------+---------------+-------------------------+
| hacluster            | stable/focal  | 2.0.3, latest/stable    |
+----------------------+---------------+-------------------------+
| hacluster            | master        | 2.0.5, latest/edge      |
+----------------------+---------------+-------------------------+
| pacemaker-remote     | stable/bionic | 1.1.x                   |
+----------------------+---------------+-------------------------+
| pacemaker-remote     | stable/focal  | 2.0.3, latest/stable    |
+----------------------+---------------+-------------------------+
| pacemaker-remote     | master        | 2.0.5, latest/edge      |
+----------------------+---------------+-------------------------+
| rabbitmq-server      | stable/bionic | 3.6                     |
+----------------------+---------------+-------------------------+
| rabbitmq-server      | stable/focal  | 3.8, latest/stable      |
+----------------------+---------------+-------------------------+
| rabbitmq-server      | master        | 3.9, latest/edge        |
+----------------------+---------------+-------------------------+
| vault                | stable/1.5    | 1.5                     |
+----------------------+---------------+-------------------------+
| vault                | stable/1.6    | 1.6                     |
+----------------------+---------------+-------------------------+
| vault                | stable/1.7    | 1.7                     |
+----------------------+---------------+-------------------------+
| vault                | master        | latest/edge             |
+----------------------+---------------+-------------------------+
| percona-cluster      | stable/bionic | 5.7, latest/stable      |
+----------------------+---------------+-------------------------+
| percona-cluster      | master        | latest/edge             |
+----------------------+---------------+-------------------------+
| mysql-innodb-cluster | stable/jammy  | 8.0, latest/stable      |
+----------------------+---------------+-------------------------+
| mysql-innodb-cluster | master        | latest/edge             |
+----------------------+---------------+-------------------------+

Trilio Charms
-------------

Trilio Charms will also need to be migrated to the charmhub delivery mechanism.
While implemented for OpenStack, the Trilio software is versioned separately
from the OpenStack releases and should use Trilio specific tracks.

+-----------------------+------------+----------------------+
| trilio-data-mover     | stable/4.0 | 4.0                  |
+=======================+============+======================+
| trilio-data-mover     | stable/4.1 | 4.1, latest/stable   |
+-----------------------+------------+----------------------+
| trilio-data-mover     | master     | 4.2 (?), latest/edge |
+-----------------------+------------+----------------------+
| trilio-dm-api         | stable/4.0 | 4.0                  |
+-----------------------+------------+----------------------+
| trilio-dm-api         | stable/4.1 | 4.1, latest/stable   |
+-----------------------+------------+----------------------+
| trilio-dm-api         | master     | 4.2 (?), latest/edge |
+-----------------------+------------+----------------------+
| trilio-horizon-plugin | stable/4.0 | 4.0                  |
+-----------------------+------------+----------------------+
| trilio-horizon-plugin | stable/4.1 | 4.1, latest/stable   |
+-----------------------+------------+----------------------+
| trilio-horizon-plugin | master     | 4.2 (?), latest/edge |
+-----------------------+------------+----------------------+
| trilio-wlm            | stable/4.0 | 4.0                  |
+-----------------------+------------+----------------------+
| trilio-wlm            | stable/4.1 | 4.1, latest/stable   |
+-----------------------+------------+----------------------+
| trilio-wlm            | master     | 4.2 (?), latest/edge |
+-----------------------+------------+----------------------+


Installation Sources
--------------------

Most charms in the OpenStack charm collection provide a config option that
allows the user to set the archive or repository that debian packages/snaps are
installed from. This is generally provided via either the ``openstack-origin``
or ``source`` charm config option, which defaults to install using the local
installations default distribution repositories.

In order to ensure that the tracks are installing the appropriate versions of
packages, the default value for this config option will be set to the
appropriate installation source for the track. If a track spans multiple distro
releases (e.g. the cloud-archive at the LTS release crossover), the charm will
need to be able to determine whether this should be sourced from the
cloud-archive or the distribution, whichever is appropriate.

Primarily, this will affect the tracks that use the Ussuri Ubuntu Cloud Archive
and the Yoga Ubuntu Cloud Archive as an installation source.

Latest Track
------------

The ‘latest’ track in charmhub is its own track, and like other tracks - charms
can be pushed to the track and the various risk levels within the track. While
it is a full blown track, to end users it is a means to get the "latest"
version of a piece of software. However, the risk level must be taken into
consideration for which version should be installed.

The latest track will be used into two ways:

* The ``latest/stable`` track will be used to park the stable 21.10 charms
  release until (at least) the 22.10 release.
* At 22.10, the ``latest/stable`` track will follow the current stable release,
  and for 22.10 this will be the ``zed`` track.

The reasoning is that the 21.10 charms' metadata advertise focal support,
against ``all`` architecturees and so it difficult to replace until a jammy,
and greater, charm can be released to the channel. However, the zed charms will
only support jammy and kinetic, and so they can be released alongside the
existing 21.10 charms. Then existing 21.10 charms won't upgrade as the new
charms will not support focal. Therefore, it is safe to re-use the
``latest/stable`` track at that point.

Risks
-----

The purpose of each risk level for each track is as follows:

Edge
~~~~

For each charm, the latest/edge channel will always map the current master
development branch.

Currently, for stable branches, the edge track will not be used, as the
Launchpad builders will push directly to the stable track. This is in keeping
with the existing policy of requiring 2 x +2 reviews on gerrit for stable
branches. At some future point, automation/policy will be introduced to enable
pushing to edge to start with and then having testing/policy to promote to more
stable risks.

Beta
~~~~

The beta risk level will be used for milestone releases during the charm
development cycle.

Candidate
~~~~~~~~~

The ``<track>/candidate`` channel will be used to track the next software
release as it enters the final weeks of development and stabilizes in
preparation for a release.  Note that this is on the next stable release track.

Stable
~~~~~~

The stable track will track the stable, tested, released version of the charm.

Technology Preview Charms
-------------------------

Charms are initially released in the "Technology Preview" state. Charms which
are in the technology preview state should not be promoted into the stable
channel until it is determined it is stable for the corresponding payload
release. Once it has graduated, all releases which have qualified as graduated
from technology preview will then be promoted to the stable channel. It is
possible that not all tracks of a charm will have a stable risk level. Charms
which are still in technology preview will only promote those charms to the
beta channel until it is determined that the charm has reached maturity, at
which point it can be promoted to the candidate and/or stable channels as
appropriate.

For example, a charm such as ovn-chassis may be delivered as technology preview
in the Stein release track but may not reach maturity until the Train release
track. The Train release track of the charm will be promoted to the stable
channel but the Stein release track will only have the charm in the beta
channel.

Special consideration is needed for those charms considered Technology Preview
at the time of this transition. Technology Preview charms are not promulgated
in the charm store and are available under the ``openstack-charmers``
namespace. Charmhub does not have namespaces in the same way and these charms
exist but are prefixed with the ‘openstack-charmers’ name, e.g.
openstack-charmers-ironic-api. These charms will be created in charmhub and
promoted to the beta channel.

Continuous Integration
----------------------

Charmhub will deliver charms that are built for specific architectures.
OpenStack charms to date, are built using python modules on amd64 architectures
and distributed as such. This is generally acceptable as they are primarily
source charms which have deliveries, but can pose some interesting challenges
for non-amd64 architectures with native extensions.

The OpenStack CI system is currently composed of a Zuul-CI system which is used
to trigger a variety of pep8 (pycodestyle), unit and functional tests for each
patch proposed to a charm. Post-merge hooks are registered which trigger a
Jenkins task to publish the charm to the legacy charm store.

Publishing Charms
-----------------

For newer charms, delivered through charmhub, they are expected to have
platform specific charms so that native extensions or tooling required by the
charm can be built for the appropriate target architecture. Launchpad now
offers charm recipes, which can be used to build and publish charms in a
similar manner to snap recipes. Changes merged into a repository will trigger a
build of the charm on all supported architectures and publish the resulting
charm to Charmhub. Leveraging this service removes a step from the OpenStack
Charmers maintained CI system for publishing charms to the charm store and
results in completely eliminating the need for jenkins in the CI
infrastructure.

The Launchpad charm recipes require that the git trees are known and stored in
Launchpad itself. In order to continue to use the upstream OpenStack gerrit
review system, which is standard for OpenStack services and has been standard
for the charms for years, each charm project in Launchpad will be configured to
mirror the source from the upstream opendev git system.

A charm recipe will be created for each branch of each charm such that upon a
patch being merged in the upstream gerrit repository, the Launchpad service
will update the mirrored (a.k.a. imported) repository, detect that new changes
are present, initiate a charm build and publish these results to the
appropriate channel(s) in charmhub.

Note that charm recipes in Launchpad will only build by default once per hour.
While there is normally a slight delay in publishing charms to the charm store,
there is one publish per change to the charm. In the Launchpad build model,
multiple changes can be queued up and included in a single build unless builds
are explicitly requested within the hour time window. This is considered
acceptable for charm development and publishing purposes.

Testing Charms
--------------

With the introduction of branches and tracks per OpenStack release, the need to
test a multitude of OpenStack releases for each change is no longer required.
Instead, targeted tests for the branch will be run. For example, a change
proposed to the Keystone charm in the stable/xena branch will run only tests
relevant to the Xena release of OpenStack.

Reducing the releases that are running in the gate allows for the charm to be
focused on the specific payload release and need not worry about spurious
failures on older releases.

The branches may also include tests that exercise the N-1 version of the
release software, in order to validate that the upgrade of N-1 to N are
functioning properly.

Charmcraft
----------

In order to leverage the charm recipes in Launchpad, all of the charms will
need to be built using the charmcraft tool. Charmcraft, however, was designed
to build and publish operator framework charms, which excludes the majority of
the charms in the OpenStack charms collection.

Charmcraft has recently grown parity with the snapcraft toolset, which allows a
variety of plugins to be used instead of the default charm plugin. The ‘dump’
plugin is appropriate to use for classic charms, however it is unsuitable for
reactive charms. In the long term, a new plugin should be created for building
reactive charms using charmcraft. In the short term, the various steps for
producing a charm can be overridden using the ‘override-build’,
‘override-stage’, and ‘override-prime’ steps which will use the existing tox
infrastructure to build, stage, and prime the reactive charm.

Documentation
-------------

Documentation for the OpenStack Charms and Ceph Charms are generally available
as a solution level, with charm README files serving as the primary source of
documentation within the Charm store and the published OpenStack Charm Guide,
OpenStack Charms Deployment Guide and Ubuntu Ceph Documentation providing
solution level documentation.

The release notes need to clearly identify the change to use and leverage
Charmhub as well as referencing migration documentation. Migration
documentation needs to be clearly outlined to make it simple for users to
migrate from using the Charm Store to Charmhub.

Developer documentation should also be provided in the OpenStack Charm Guide to
provide details for developers wishing to contribute new features, bug fixes,
etc.


Alternatives
------------

There aren't really any good alternatives. When the charmstore to charmhub
cut-over is implemented, the existing ``charm`` command will stop working,
which means that the entire build and publishing system in Jenkins will simply
stop working too. So the only real alternative is to re-work the Jenkins
software to use charmcraft and then use that to push to the charm store.

This means continuing to maintain Jenkins and not take advantage of the
Launchpad builders to build architecture aware charms.

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  Alex Kavanagh

Gerrit Topic
------------

N/A

Work Items
----------

Produce 'charmcraft' recipes for classic and reactive charms
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Launchpad can be used to build charms when branches in git repositories are
changed. These builds are controlled by recipes. Part of the work will be to
ensure that these recipes work for all of the charms. They will be centrally
controlled and 'synced' into charms as part of the development cycle.

* Write classic recipe
* Write reactive charm recipe using charm-tools ``build`` command.
* Add recipes to ``release-tools`` repository.

Produce helper tools/library to manage launchpad projects/recipes
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In order to manage the charms, recipes and mapping of git repository branches
to charmhub tracks/channels, a suite of tools and configuration will be
produced to manage the complexity of the number of charms, branches and tracks.

Essential functionality will be:

* display gap between config and actual in git repositories and launchpad
  recipes.
* Show desired tracks vs actual tracks for charms (in charmhub) to generate
  'requests' for new tracks, or for tracks to be deleted.
* update existing recipes against the configuration.
* Create new recipes as required against the config.


Repositories
------------

Initially, the existing ``release-tools`` repository will be the collecting
point for the tools, but it has become a bit of a dumping ground for related
tools and needs a major re-organistion. In the fullness of time, these tools
may be broken out into their own repository where changes to the config
automatically updates the recipes.

Additionaly, the manage the recipes a separate ``charmhub-lp-tools`` tool will
be created to automate recipe creation, updates, deletes and changed, and to
authorize uploads to the charmhub.

.. code-block:: bash

    https://github.com/openstack-charmers/release-tools
    https://github.com/openstack-charmers/charmhub-lp-tools

Documentation
-------------

Documentation for the tools, configuration schemas and how the system works
will be inside the repository in the form of Markdown files. This is to ensure
that the documentation 'lives' with the code. Where possible documentations
for the schemas (config files) will be in the config files themselves.

Security
--------

No additional security concerns.

The user that runs the tool must have a launchpad account in the
``openstack-charmers`` team, and a charmhub account in the
``openstack-charmers`` team.

Testing
-------

The possibility to break the charmstore and charmhub is pretty high, so the
tools and process will be developed incrementally against 'test' charms and low
priority 'real' charms.

Stage 1
~~~~~~~

Initially copies of the existing charms will be registered into the charmhub
(under new names), to test the recipes and ability to build charms. This is to
verify that the charm build recipes work for classic and reactive charms.

Stage 2
~~~~~~~

Selecting two charms (notionally, ``neutron-dynamic-routing`` and
``manila-generic``), break the link to the charmstore and ensure that tracks,
branches and recipes can be created such that updates to the charms git
repositories result in built charms in the correct tracks.

This will allow the correct changes to the charms to be enumerated and captured
in the release-tools repository.

Dependencies
============

None

References
----------

* `Charmhub unofficial docs <https://github.com/canonical/charmhub-unofficial-docs>`__
* `Snapcraft channels <https://snapcraft.io/docs/channels>`__
* `Mirroring repositories from other sites <https://help.launchpad.net/Code/Git#Mirroring_repositories_from_other_sites>`__

.. targets
.. _`Snap store`: https://snapcraft.io/docs/channels
