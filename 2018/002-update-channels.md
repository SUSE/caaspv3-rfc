 Title

| Field  | Value      |
|:-------|:-----------|
| Status | Draft      |
| Date   | 2018-07-31 |

## Introduction

### Problem description

Updates can be disruptive for a customer's application stack (ex. Kubernetes
upgrade from 1.10.x to 1.11.x, default pod security policy introduction).

Right now there is no visibility of what is inside an update and furthermore
there is no possible way around intrusive updates by having the ability to only
apply critical bug fixes and security updates. An admin needs to apply all updates
that come through a single channel.

In order to separate intrusive updates in SUSE CaaS Platform, we need different
update channels. This allows the separation of possibly disruptive updates
into their own channels, and lets the admin continue applying bugfixes and security
updates without the need to apply intrusive, new features. The admin can also see
what is inside an update to be able to decide when to apply it.

The channels will also act as a barrier to avoid that a user runs k8s 1.8 and
upgrades right away to 1.11 in one step.

### Concrete Example

SUSE CaaS Platform installations will still have two repositories active. The Update and
the Pool Channel. Both have the same version. This repositories, hereinafter called "channel"
are associated with a certain version of the product, for example: `3.1`.
The update channel is going to provide regular maintenance updates (eg: security fixes) and bug
fixes for the specific product codebase associated with the channel. Invasive updates are
going to be shipped to a new different channel. This will result in a minor version of the
product to be announced.

Once a new channel is published, customers will have a choice between moving to the new one,
or stay on the current one. For X months the old channel will get security fixes and bug
fixes, that do not require intrusive changes. Once this time is over, the customer can stay
registered against the channel but won't get any update at all (even security fixes) and will
lose support.

To make a concrete example:
1. The current version of CaaSP is 3.0, this consists of two channels (3.0 pool + 3.0 updates)
    1. `bash` security fix is sent to the 3.0 updates channel
    1. `kubernetes-salt` bug fixes are sent to the 3.0 updates channel
1. A new version of Kubernetes is released, this consists of two channels (3.1 pool + 3.1 updates)
    1. The 3.1 pool channel containing the new kubernetes version is created
    1. A ssl security fix is pushed both to the 3.0 and to the 3.1 updates channel
    1. `kubernetes-salt` bug fixes are sent to the 3.1 updates channel and eventually to 3.0 updates
1. *X* months passed since the creation of the 3.1 release
    1. A kernel security fix is pushed **only** to the 3.1 updates channel.
       Nothing is pushed to the 3.0 updates channel anymore.

### Proposed change

The admin will have the ability of choosing when to switch to the next channel.

In case of an airgapped system, a link to the release notes of the next update channel
is displayed. A system with access to the web can fetch the release notes and
display them in the UI directly. In both scenarios, an admin can evaluate the
upcoming changes beforehand.

The UI of velum needs to be adapted to enable a switch to a new channel.

## Detailed RFC

We can separate the updates into different channels. There are 2 types of channels:

* update channel: normal maintenance updates, which are safe and non-intrusive
* new version channel: version updates which can contain intrusive updates

This RFC proposes:

* a front-end change in the velum UI to enable the admin to switch to a new version
channel
* a back-end change in salt to perform the channel switch on all cluster nodes

### Phase 1 (1st iteration)

#### Tools/Services

We need a set of tools/services to indicate, whenever a maintenance update or a new version
update is ready to be installed. They should run on each worker node to be more independent
and decoupled from velum. If velum would be the initiator of the update check, it might lead
to inconsistent and long execution runs in larger clusters.

##### "check for updates" service on each worker

The current channel will offer maintenance updates, which need to be announced.

The service:

* runs once per day e.g.
* is generally decoupled from velum, runs independently, and just reports.
* reports availability of maintenance updates to velum via grains.

##### "new version" checker service on each worker

The next channel will offer possibly disruptive updates, which can be either features or
version updates of certain components of the whole stack, which need to be announced.

The service:

* runs once per week or longer timeframe.
* is generally decoupled from velum, runs independently, just reports.
* checks for new versions (migration targets):
  - query SCC API if there is a new compatible version available for this CaaSP version (SUSEConnect library)
  - reports availability of new versions to velum via grains.

  Furthermore if the system is registered against SMT/RMT:
  - query RMT/SMT API if there is a new compatible version available for this CaaSP version
  - reports availability of new versions to velum via grains.
  - reports if an admin needs to mirror the new version channel completely first (out of sync mirror)
* get the link to the release notes

also see section "Systems Registered against RMT/SMT"

#### User Interface

The user interface of velum needs to be adapted minimally.

* indirect display of new version content through a link to the release notes
* button to switch to the next channel in velum
  - the button will trigger `transactional-update salt migrate`
* confirmation of the user that the mirrors are synced
  - this confirmation (button) will trigger the "new version" checker again to make sure the sync is done and grains are updated

Velum needs to know when a new version is available, by looking at a special grain, that indicates
the availability of a new version. This grain is set by the service that runs on each worker.

#### Salt back-end

##### Triggering updates/migrations

We will continue to use `transactional-update` to trigger the updates and migrations.
Salt will be used to call `transactional-update migrate`

We can reuse the update orchestration with an extra option to call `transactional-update migrate`.

`transactional-update` will report the result back to velum via grains.

##### Updating in batches

Updating in batches reduces the chances that salt might timeout, and relaxes the network connection.

As clusters can have a large number of nodes, it would create a bottleneck if we run the migration
on all nodes at once, due to network congestion because of the download of the packages, which can
be quite large.

Therefore we should have a logic in salt that runs the migration on a set (fixed number) of nodes
at a time, before going to the next batch.

Salt will have to actively wait for the migrations to finish, and then set the specific grain to
done once it finished.

#### General considerations for Phase 1

##### Out of sync nodes

There needs to be a way to get a single node up-to-date with maintenance updates and
new migration targets, especially when a single node is out of sync with updates, due to
having been powered off for a longer time, or if the customer added a new node from
an old ISO image to the cluster.

##### Systems Registered against RMT/SMT

If the system is registered against SMT or RMT, we can access the information, that there
is a new version, but that this version is not completely mirrored to SMT/RMT.
So we need an additional information: a new version is available, but the admin needs to
mirror this first on SMT/RMT.

### Phase 2 (2nd iteration)

#### User Interface

* explicit trigger of maintenance update installation (snapshot creation and reboot)
  - before that change, a user can only trigger the reboot, snapshot/installation happens automatically
* direct display of update content in velum through parsing the release notes
* display of update history via velum

#### Manual/automatic update strategy

A customer can choose conservative or progressive update strategy:

* Run maintenance updates manually
* Run maintenance updates automatically, but switch channels manually
* Run maintenance updates and switch to update channels automatically (always bleading edge)

#### General considerations for Phase 2

##### Being outdated before a migration

We need to check if the installed packages are good enough to do the migration or not.
If it's good enough, we can either continue with the migration directly or apply normal
updates and prolong the migration.
The user should not be enforced to run all maintenance updates first, because the migration
would contain/overwrite the patches anyways.

### Phase 3 (3rd iteration)

#### Release notes for mainenance channel updates

It would be also good to know the exact changes of a maintenance update, not only channel
updates.

#### API

Furthermore there could be an API available through velum to enable easy scripting

* endpoints:
  - GET:  list current channel maintenance updates (non disruptive)
  - POST: trigger current channel maintenance updates
  - GET:  list next channel (new version) updates
      - A: disruptive update in the same caasp product/version (e.g. k8s minor version bump, container runtime)
      - B: migration target once overlap support is EOL (next caasp product/version)
  - POST: switch to the next channel
  - GET:  list update history
* we would need a way of CLI authentication to get to this API

### Dependencies

* new channels can be created on demand in SCC

### Concerns and Unresolved Questions

#### General

* categorize updates
  - when to put an update in a separate channel?
    - adhoc (as needed by specific package)
  - how long is the overlapped support for a channel?
    - couple of months

## Alternatives

None known at the moment.

## Revision History:

| Date       | Comment       |
|:-----------|:--------------|
| 2018-07-31 | Initial Draft |
| 2018-08-30 | Update for more mirror sync details |
