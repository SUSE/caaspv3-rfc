Helm chart repository RFC

| Field  | Value      |
|:-------|:-----------|
| Status | Draft      |
| Date   | 2018-07-16 |

## Introduction

> This section targets end users, operators, PM, SEs and anyone else that might
need a quick explanation of your proposed change.

Kubernetes is becoming a successful platform used to deploy different kind of workloads.

Deploying workloads on top of Kubernetes requires users to download Kubernetes manifest files and apply them; this can be tedious and error prone.

[Helm](https://helm.sh/) is the most common way to deploy and maintain containerized workloads on top of Kubernetes.
Helm works by having a so called “repository” that includes several “charts”, one per workload. Each chart describes the Kubernetes resources required to run the workload.

Drawing an analogy with the RPM world, charts are a bit like packages, a helm chart repository is like a package repository and, finally, `helm` is a package manager like `zypper` that operates using charts instead of packages.

At SUSE some of our products have been already created/changed to run on top of Kubernetes or are about to be ported. SUSE chose to distribute its products using Helm.
SUSE is maintaining the helm charts required to install its products and is serving them from a SUSE operated helm chart repository.


### Problem description

> Why do we want this change and what problem are we trying to address?

As mentioned above, SUSE has a helm chart repository that contains the charts for its own products.
All these products are using container images shipped by the SUSE container
registry.

The current solution works but doesn’t address the following problems:

  * Ensure all the images referenced by the helm charts are available on the SUSE
container registry and are kept in sync. Ideally new charts and images should be published at the same time to avoid the risk of having charts referencing not yet available images.
  * Provide an offline copy of the SUSE chart repository to be used by customers
installing our products into air gapped environments. That includes providing a way to integrate with mirrored registries
  * Ensure the workflow fits within the requirements of the SUSE Maintenance, Maintenance QA and security teams.
  * Have reference copies of supported upstream charts that are working with SUSE CaaS
      Platform. For example the ingress helm chart for nginx from upstream is currently working
      with latest version of SUSE CaaS Platform. However future changes to the chart upstream might break it.

### Proposed change

We should build a container image that runs a helm chart repository pre-populated
with all the SUSE charts.

This container image will be hosted on the SUSE registry and will be mirrored
inside of the on-premise container registry of our customers, like it happens
with all the other images we are shipping.

Customers operating inside of an air-gapped environment will have to:

  * Ensure the contents of the SUSE registry are mirrored locally
  * Deploy an instance of our offline chart repository image (ideally on top
    of a kubernetes cluster)
  * Configure their helm clients to point to the local instance of the
    offline chart repository

Once these steps are done they will be able to deploy any kind of "helm shipped"
product without having their nodes reach the outer network.

## Detailed RFC

> In this section of the document the target audience is the dev team. Upon
> reading this section each engineer should have a rather clear picture of what
> needs to be done in order to implement the described feature.

### Proposed change (Detailed)

#### Contents of a chart repository

A helm chart repository is a collection of `.tgz` files containing the charts
and a `index.yaml` file referencing them. The charts are also called *"packages"*.

The helm chart repository can also include *provenance* files. These are used to
verify the integrity and origin of a package.
Provenance files are not mentioned by the `index.yaml` file. The client will
look for them at the `<package name>.tgz.prov` location.

A read-only chart repository (like the one we are interested in) can be served by a simple
web server.

More information about the provenance files can be found
[here](https://docs.helm.sh/developing_charts/#helm-provenance-and-integrity).

More information about the format of the index file can be found
[here](https://docs.helm.sh/developing_charts/#the-index-file).

#### ChartMuseum

[ChartMuseum](https://github.com/kubernetes-helm/chartmuseum) is a fully working
helm chart repository. It doesn't just simply host read only chart repositories, it also allows users to upload, update and delete them.

We plan to offer ChartMuseum to our customers so that they can use it to host
their own charts.

ChartMuseum can also be used to host a simple read-only helm chart repository. That, combined with the ability to generate the `index.yaml` file in a dynamic fashion, makes
ChartMuseum a viable solution to implement an offline chart repository.

A simple POC can be created on a docker host by following these steps:

* Create a `charts` directory on your docker host.
* Put some chart files (the `.tgz` ones) into this directory.
* Run the [https://hub.docker.com/r/chartmuseum/chartmuseum/](chartmuseum/chartmuseum) docker image.

For example, given the following directory `/home/flavio/suse-charts`:

```
$ ls -l /home/flavio/suse-charts
total 388
-rw-r--r-- 1 flavio users 89627 Jul  3 10:00 cf-2.0.2.tgz
-rw-r--r-- 1 flavio users 32344 Jul  3 10:00 cf-2.10.1.tgz
-rw-r--r-- 1 flavio users 24001 Jul  3 10:00 cf-2.4.1-beta3+cf278.0.g12f5d3a8.tgz
-rw-r--r-- 1 flavio users 20567 Jul  3 10:00 cf-2.5.0-beta4+cf278.0.gafa3d0e9.tgz
-rw-r--r-- 1 flavio users 21575 Jul  3 10:00 cf-2.6.11.tgz
-rw-r--r-- 1 flavio users 21589 Jul  3 10:00 cf-2.6.1-rc1+cf278.0.g52d7a644.tgz
-rw-r--r-- 1 flavio users 21645 Jul  3 10:00 cf-2.7.0.tgz
-rw-r--r-- 1 flavio users 26401 Jul  3 10:00 cf-2.8.0.tgz
-rw-r--r-- 1 flavio users  2268 Jul  3 10:00 cf-usb-sidecar-mysql-0.0.0-366-gcf3d373.tgz
-rw-r--r-- 1 flavio users  2271 Jul  3 10:00 cf-usb-sidecar-mysql-0.0.0-pre.35-develop.tgz
-rw-r--r-- 1 flavio users  2257 Jul  3 10:00 cf-usb-sidecar-mysql-1.0.1.tgz
-rw-r--r-- 1 flavio users  2500 Jul  3 10:00 cf-usb-sidecar-postgres-0.0.0-366-gcf3d373.tgz
-rw-r--r-- 1 flavio users  2507 Jul  3 10:00 cf-usb-sidecar-postgres-0.0.0-pre.23-develop.tgz
-rw-r--r-- 1 flavio users  2489 Jul  3 10:00 cf-usb-sidecar-postgres-1.0.1.tgz
-rw-r--r-- 1 flavio users  4467 Jul  3 10:00 console-0.9.7.tgz
-rw-r--r-- 1 flavio users  3878 Jul  3 10:00 console-0.9.8.tgz
-rw-r--r-- 1 flavio users  3880 Jul  3 10:00 console-0.9.9.tgz
-rw-r--r-- 1 flavio users  3871 Jul  3 10:00 console-1.0.0.tgz
-rw-r--r-- 1 flavio users  3873 Jul  3 10:00 console-1.0.2.tgz
-rw-r--r-- 1 flavio users  4078 Jul  3 10:00 console-1.1.0.tgz
-rw-r--r-- 1 flavio users 11243 Jul  3 10:00 uaa-2.0.2.tgz
-rw-r--r-- 1 flavio users  6180 Jul  3 10:00 uaa-2.10.1.tgz
-rw-r--r-- 1 flavio users  6988 Jul  3 10:00 uaa-2.4.1-beta3+cf278.0.g12f5d3a8.tgz
-rw-r--r-- 1 flavio users  4152 Jul  3 10:00 uaa-2.5.0-beta4+cf278.0.gafa3d0e9.tgz
-rw-r--r-- 1 flavio users  4143 Jul  3 10:00 uaa-2.6.11.tgz
-rw-r--r-- 1 flavio users  4175 Jul  3 10:00 uaa-2.6.1-rc1+cf278.0.g52d7a644.tgz
-rw-r--r-- 1 flavio users  4144 Jul  3 10:00 uaa-2.7.0.tgz
-rw-r--r-- 1 flavio users  5470 Jul  3 10:00 uaa-2.8.0.tgz
```

A simple read-only chart registry can be created with the following command:

```
docker run --rm \
  -v /home/flavio/charts:/charts:ro \
  -p 4000:4000 \
  chartmuseum/chartmuseum \
    --port 4000 \
    --disable-api \
    --storage local \
    --storage-local-rootdir /charts
```
The above command mounts the charts directory from the host into the container
using the read-only mode. Then it starts ChartMuseum with the following flags:

* `--disable-api`: all the write operations (create, update, delete) are
  disabled, only read operations are allowed.
* `--storage` and `--storage-local-rootdir`: all the charts are stored on the
  local filesystem and are inside of the `/charts` directory.

The `index.yaml` file can be obtained by executing: `curl http://localhost:4000`.
The file is generated dynamically by ChartMuseum at boot time and is kept
in memory.

#### The offline chart repository image

SUSE already plans to distribute a ChartMuseum image to all its customers. The
purpose of this is image is to allow them to run their own internal chart
repository where they can upload their personal charts.

To accomplish that SUSE will create and maintain a container image of
ChartMuseum. During the rest of this RFC we will address this image as
`registry.suse.com/chartmuseum`.

The [entrypoint](https://docs.docker.com/engine/reference/builder/#entrypoint)
of this image will start ChartMuseum in "normal" mode, meaning all the API endpoints will be enabled (making the repository read-write).


The SUSE chart repository available at [kubernetes-charts](https://kubernetes-charts.suse.com)
is going to be shipped inside of an image called
`registry.suse.com/suse/kubernetes-charts` (again, the name is just an example
chosen for this RFC, the final name might change).

The `registry.suse.com/suse/kubernetes-charts` image is going to be based
on the `registry.suse.com/chartmuseum` image. It will extend it by:

  * Adding the RPM packages providing the helm charts of SUSE products
  * Providing a new entrypoint that makes the registry read-only and points
    ChartMuseum to the location where all the SUSE charts have been
    installed by the RPM.

Ideally each SUSE team will ship their own charts via a single RPM per chart. All the
charts will be installed into the same location, for example `/usr/share/suse-kubernetes-charts`.

For example the SUSE CaaS Platform team will provide its own RPMs
that, once installed, will create the following files:

  * `/usr/share/suse-kubernetes-charts/ingress-1.1.0.tgz`
  * `/usr/share/suse-kubernetes-charts/ingress-2.0.0.tgz`
  * `/usr/share/suse-kubernetes-charts/monitoring-2.0.1.tgz`
  * `/usr/share/suse-kubernetes-charts/dashboard-3.0.0.tgz`

The SUSE Cloud Application Platform team will provide a set of rpms that,
once installed, will create the following files:

  * `/usr/share/suse-kubernetes-charts/cf-2.0.2.tgz`
  * `/usr/share/suse-kubernetes-charts/cf-2.10.1.tgz`
  * `/usr/share/suse-kubernetes-charts/cf-2.6.11.tgz`
  * `/usr/share/suse-kubernetes-charts/cf-2.7.0.tgz`
  * `/usr/share/suse-kubernetes-charts/cf-2.8.0.tgz`
  * `/usr/share/suse-kubernetes-charts/cf-usb-sidecar-mysql-1.0.1.tgz`
  * `/usr/share/suse-kubernetes-charts/cf-usb-sidecar-postgres-1.0.1.tgz`
  * `/usr/share/suse-kubernetes-charts/console-0.9.7.tgz`
  * `/usr/share/suse-kubernetes-charts/console-0.9.8.tgz`
  * `/usr/share/suse-kubernetes-charts/console-0.9.9.tgz`
  * `/usr/share/suse-kubernetes-charts/console-1.0.0.tgz`
  * `/usr/share/suse-kubernetes-charts/console-1.0.2.tgz`
  * `/usr/share/suse-kubernetes-charts/console-1.1.0.tgz`
  * `/usr/share/suse-kubernetes-charts/uaa-2.0.2.tgz`
  * `/usr/share/suse-kubernetes-charts/uaa-2.10.1.tgz`
  * `/usr/share/suse-kubernetes-charts/uaa-2.6.11.tgz`
  * `/usr/share/suse-kubernetes-charts/uaa-2.7.0.tgz`
  * `/usr/share/suse-kubernetes-charts/uaa-2.8.0.tgz`

The same thing will be done by the other teams at SUSE who want to ship containerized
workloads managed by helm.

#### Building the offline chart repository image

The `registry.suse.com/suse/kubernetes-charts` will be built inside of the
Open Build Service (obviously inside of IBS).

The build service project will consists of:

  * RPM packages of helm charts.
  * RPM defining patterns of helm charts.
  * Definition of the `registry.suse.com/suse/kubernetes-charts` container image.

The next sections will provide details about these resources.

##### RPM packages definitions

Each helm chart should have its own RPM package. For example we would have a
`cf-chart`, `console-chart`, `uaa-chart`, …

We must be able to ship multiple versions of the same helm chart (eg: `cf-2.0.2.tgz` and `cf-2.10.1.tgz`). To achieve that each RPM will install one `.tgz` file per version of the chart.

For example, the build service package for the `cf-chart` RPM would consist of the following files:

  * `cf-chart.spec`
  * `cf-chart.changelog`
  * `cf-2.0.2.tgz`
  * `cf-2.10.1.tgz`

Once installed the RPM would provide the following files:

  * `/usr/share/suse/kubernetes-charts/cf-2.0.2.tgz`
  * `/usr/share/suse/kubernetes-charts/cf-2.10.1.tgz`

In the future each chart RPM should also provide the `.tgz.prov` provenance files. However this RFC doesn't want to cover the problem of creating these signed files (currently they are missing also from *”kubernetes-charts.suse.com”*).

Each team would retain ownership of these packages. That would allow them to move at
their own pace and not rely on other teams to process their submission requests.

##### RPM pattern definitions

Each team will have its own RPM defining a pattern that requires all the single
helm charts that are related with it.

For example the Cloud Application Platform team could have a `cap-charts`
pattern that requires the different packages providing the cloud foundry charts (eg: `cf-chart`, `uaa-chart`, ...).

Each team would retain ownership of their pattern packages.
That would allow them to move at their own pace and not rely on other teams to
process their submission requests.

##### Container image definition

The OBS project would include also the definition of the `registry.suse.com/suse/kubernetes-charts` container image.

This image would take the ChartMuseum image maintained by the SUSE CaaS Platform
team and extend it:

  * The image would have a custom entrypoint that starts ChartMuseum with the right
set of flags.
  * The image would require the different RPM patterns defined inside of
the project. By doing that each pattern will automatically pull all the
individual chart packages into the final image.

The usage of the RPM patterns allows the image definition to stay lean and
greatly reduces the amount of changes to perform against it.

Finally, by using RPMs to ship the charts, we will be 100% sure multiple teams are not going to provide conflicting `.tgz` files (as in the same helm chart being provided by two different teams). Such a situation will be immediately result in a package conflict and prevent the final container image from being created.

#### Maintenance operations

This is an overview of the possible operations related with the helm charts and how these operations would translate against the open build service.

  * Adding new helm charts: a new RPM for the chart is created. One of the RPM
    pattern is updated to reference this new chart package. No change is needed
    inside of the container image definition; the build service will automatically
    rebuild it.
  * Upgrading an existing helm chart: the RPM defining the helm chart is updated,
    no change is needed inside of the RPM pattern requiring it, no change is
    required inside of the container image definition. OBS will automatically
    rebuild the image. Note well: adding a new version of an helm chart
            falls into this category.
  * Removing a chart: the RPM defining the chart has to be removed from the
    project. The RPM pattern referencing the chart is updated. No change
    is needed inside of the container image definition. OBS will automatically
    rebuild the image.
  * Add a new helm chart pattern: a new RPM defining the pattern is added to
    the project, probably together with a bunch of new packages providing the
    charts referenced inside of it. The container image definition is updated
    to require this new pattern. OBS will automatically rebuild the image.
  * Remove a helm chart pattern: the RPM pattern is removed from the project
    together with all the packages that were being referenced by it. The
    container image is going to be updated to remove the reference of the
    pattern. OBS will automatically rebuild the container image.

In each case it will be necessary to perform a OBS submission request from the
devel project against the target project used by Maintenance QA. From this final
project the `suse/kubernetes-charts` image is going to be published on the SUSE
container registry.

If the helm chart is referring a new set of container images, the submission
request can include also the new container images. By doing that we ensure the
helm charts and the contents of the SUSE registry are always aligned; we do not
risk of having a helm chart referencing a container image that has not yet been
published on the SUSE registry.
**TODO:** is this possible? I mean having a SR that includes packages from one
project (the devel prj of charts) and another one (the prj where the image
referenced by the new chart is built).

#### Automation

The `.tgz` files can be easily created by running the `helm package` command
against a checkout of the helm chart source code.

This is a proposed workflow to automate the whole process of writing
charts and shipping them into the `registry.suse.com/suse/kubernetes-charts`.

  * Charts owned by a team will live into their own dedicated git repository
  * Changes to the charts will happen by using a pull request mechanism
  * The team owning the chart repository can integrate an external CI system and
    trigger the execution of tests against the pull request.
  * Once the changes are merged into master a pipeline is started that performs the following actions:
    * Run `helm package` against the latest version of the git repository. This will produce the `.tgz` file of the latest package.
    * The `.tgz` package is automatically added to the build service package defining the RPM
    * The spec file and changelog of the RPM are automatically updated to reflect the presence of
       the new `.tgz` file.
  * The update of the chart RPM will trigger a rebuild of the chart container image.

Updating the chart image can be done in a similar way to how the SUSE CaaS
Platform teams implements their automated packaging pipeline:

  * Multiple concourse jobs are defined for each chart package
  * Each concourse job monitors the changes submitted to the master branch of
    the git repository containing the source code of the chart
  * The OBS package is automatically updated by concourse

This will automatically trigger a rebuild of the RPM package of the chart and
later of the chart container image.

**Note well:** it would also be possible to build helm charts based on openSUSE inside of the OBS instance and then have them synchronized inside of IBS to build their SLE counterparts. This is similar to what the SUSE CaaS team is doing with other bits and pieces of Kubic.

#### Online helm chart repository

The SUSE Customer Center team is currently maintaining the hosted version of the SUSE chart repository.

This is currently implemented by exposing the contents of a S3 bucket. The S3 bucket contains all the `.tgz` files and the `index.yaml` that make the repository. The helm chart creation, the index generation and the upload to S3 is currently automated via a concourse pipeline described [here](https://github.com/SUSE/cloudfoundry/wiki/Building-SUSE-Cloud-Application-Platform#publishing-to-the-helm-repository-httpskubernetes-chartssusecom).

If this proposal is accepted the existing automation pipeline could be extended to:

  * Download the latest version of the SUSE helm chart image: `skopeo copy docker://registry.suse.com/suse/kubernetes-charts:latest`. oci:///tmp/automation/suse-kubernetes-charts:latest`.
  * Extract the contents of the image: `umoci unpack --rootless --image /tmp/suse-kubernetes-charts:latest /tmp/automation/suse-kubernetes-charts-unpacked`.
  * Copy the helm charts shipped with the image: `mv /tmp/automation/suse-kubernetes-charts-unpacked/rootfs/usr/share/suse-kubernetes-charts/ /tmp/automation/`.
  * Recreate the `index.yaml` file: `helm repo index /tmp/automation/suse-kubernetes-charts`.
  * Upload all the contents of `/tmp/automation/suse-kubernetes-charts` to S3.

Note well: neither `skopeo` nor `umoci` require access to a docker daemon, hence they can be run inside of a container in an easy fashion.

It would also be possible to not use the S3 bucket at all and just run an instance of the latest version of the `registry.suse.com/suse/kubernetes-charts` container image.



### Dependencies

> Highlight how the change may affect the rest of the product (new components,
> modifications in other areas), or other teams/products.

The adoption of this workflow would have an impact on the current online helm chart repository. This has been outlined in the previous section (see *“Online helm chart repository”*).

### Concerns and Unresolved Questions

> List any concerns, unknowns, and generally unresolved questions etc.

#### How do our user update the local helm chart container image

How does our users become aware of a new version of the SUSE offline helm chart repository being available?
This is part of a bigger topic that involves all the images. It’s going to be addressed by a separate RFC.

#### How to handle automatic rebuilds of images referenced by Helm charts

All our images are built into the build service and can be rebuilt automatically every time one of their packages is updated.

What should happen to the tag of the image? How can we prevent existing helm chart to become broken or even worse obsolete (as in pointing to an outdated and vulnerable image).

This problem is out of the scope of this RFC. We will elaborate on it inside of another RFC (probably the same one mentioned above).

#### Products with different requirements of the same chart

Multiple products might rely on other containerized software shipped via helm. For example we could imagine the `cf` and the `portus` helm charts requiring the `nginx-ingress` chart.

If `cf` and `portus` require two different versions of the `nginx-ingress` controller we can envision two possibilities:

  * Have `cf` and `portus` will have the right version of `nginx-ingress` as a [subchart](https://docs.helm.sh/chart_template_guide/#subcharts-and-global-values)
  * The `nginx-ingress` RPM will ship multiple `.tgz` files ensuring both the version required by `cf` and the one required by `portus` are available.

The second option seems the best approach.

## Alternatives

> List any alternatives considered, and the reasons for choosing this option
> over them.

The SUSE Cloud Application platform is already shipping helm charts. The process
they are using relies on an automated pipeline built inside of concourse.

The process doesn't take into account the offline use case and does not follow the SUSE maintenance processes which many other product teams at SUSE follow.


## Revision History:

| Date       | Comment       |
|:-----------|:--------------|
| 2018-07-16 | Initial Draft |

