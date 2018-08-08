# Containers Tagging Strategy

| Field  | Value      |
|:-------|:-----------|
| Status | Draft      |
| Date   | 2018-08-8  |

## Introduction

As SUSE starts delivering software shipped in containers accessible
from the [SUSE container registry](https://registry.suse.de). Images within the
registry are referenced by a name and a tag or digest. The digest
is a sha256sum of the image binary (thus assumed to be a unique identifier
of an image) and tags are a valid ASCII name (uppercase & lowercase, digits,
dashes and underscores).

Pulling from a registry with the docker client looks like:

```
docker pull <registry-domain>/<name>:<tag>
```

or 

```
docker pull <registry-domain>/<name>@<image-digest>
```

SUSE already started to deliver some applications wrapped in containers.
Initially the containers have been delivered wrapped into RPM package, but now
containers will be delivered using the SUSE container registry.

Making an analogy with applications delivered with RPMs, the SUSE container
registry can be seen as the online rpm repositories, the name as the package
name and tags as package versions.

Tags are just text identifiers of an image, they aren't meant to be comparable
or ordered. Assuming tags can just work as versions do for packages is just
inaccurate. Containers in the registry have no versions neither releases, they
only have tags, text identifiers. Relevant to note that each image in the
repository can be referenced by multiple tags. 

Also a complete refefence (`<registry-domain>/<name>:<tag>`) always
points to a unique image.

## Problem description

Kubernetes pulls the images referenced in the Kubernetes manifest. Kubernetes
checks if those images are available locally and if not pulls them from the
registry. It means that if an image reference is overwritten with the same
reference Kubernetes will not pull it again and then images of a running
cluster start to diverge from the images available into the registry.

The above situation could be specially tricky when, for instance, new nodes are
added to a cluster that is running silently with outdated images. This is
particularly tricky because it can easily end up in a situation where the
cluster is running multiple versions of the same image. Let's imagine the
cluster was running some outdated mariadb image, then when a new node is added
and this node pulls the images from the registry (including the mariadb one)
it will pull and run different versions of the images without noticing it.
Running a cluster with multiple mariadb versions without control or being
enforced by the administrator is clearly dangerous and painful to debug if it
causes some issue at some point.

The tagging strategy of the delivered images must be aligned with the build and
update workflow. Otherwise it is pretty simple to end up in similar situations
as exposed above.

## Current situation

Before SUSE container registry images have been and are being delivered wrapped
in RPMs. This way the update and delivery worklfow is the same as any other
packages. However within the containers ecosystem this presents some isssues:

* Containers ecosystems based on Kubernetes are not disigned to interact with
  RPM repositories but container registries.
  
* Installing, updating and uninstalling RPMs can be costly. In CaaSP it
  requires a transactional update.
  
* Using RPMS requires additional services to handle images upload into docker
  or crio daemons. See container-feeder tool or sle2docker. While systems like
  Kubernetes are already prepared to pull and load images from a registry.
  
* Big downloads, RPMs do not benefit from layers reuse. RPMs contain all
  container layers even only unique layers are finally loaded into the daemon. 
  
In this context stable tags for references are being used. For CaaSP images are
tagged with a fixed tag (usually the version of the main containerized
application) defined in the kiwi file description and not modified during
updates, however the RPM includes a metadata file used by the container-feeder.
At loading time container-feeder re-tags the image with the '<tag>-<release>'
and 'latest' tags, this way, even using stable tags in manifests the images
are locally tagged with multiple tags, including the specific release tight to
an RPM and image build.

## Proposed change

Introduce an app or service to track updates in the registry, something similar
as what zypper does for RPMs and repositories. This is a key point and the
proposed strategy here assumes there is going to be something like that
available.

The proposed change mimics the tagging strategy exposed above when using images
wrapped in RPMs, but using the SUSE container registry and OBS instead.

`kiwi` and `skopeo` tools can build images with multiple tags (since versions
9.15.3 and 1.30.0 respectively). Thus the main idea is to keep using stable
tags to facilitate image references in manifests but also tag images in
registry with additional dynamic tags, based on included packages versions or
image release numbers. This way we could have in the registry multiple images
with a meaningful tags pointing to a specific version, but also a stable tag
pointing to the latest version of an image.

Imagine there is a mariadb image tagged as below in the registry (emulating
`docker images` output):

```
REPOSITORY           TAG                    IMAGE ID
caasp/mariadb        10.2                   1ade07d13d13
caasp/mariadb        latest                 1ade07d13d13
caasp/mariadb        10.2.15-3.2            1ade07d13d13
```

Note that the build ID is being used as the release number. Build ID are
the build numbers currently included in the resulting files of KIWI build
within the build service.

All three tags pointing to the same image. Then if a new update due to a new
mariadb release happens adding the new image could look like that:

```
REPOSITORY           TAG                    IMAGE ID
caasp/mariadb        10.2                   70b5d81549ec
caasp/mariadb        latest                 70b5d81549ec
caasp/mariadb        10.2.15-3.2            1ade07d13d13
caasp/mariadb        10.2.15-3.3            70b5d81549ec
```

Note that old image would be still accessible with the concrete version tight
to the specific version and release. Again, if a new update happens but this
time due to a new mariadb version adding a third image into the registry could
result in:

```
REPOSITORY           TAG                    IMAGE ID
caasp/mariadb        10.2                   70b5d81549ec
caasp/mariadb        latest                 284549eacf84
caasp/mariadb        10.2.15-3.2            1ade07d13d13
caasp/mariadb        10.2.15-3.3            70b5d81549ec
caasp/mariadb        10.3                   284549eacf84
caasp/mariadb        10.3.1-3.4             284549eacf84
```

Aging all three image versions are accessible and there is a stable reference
to get the latest. However this time, since the mariadb version changed, there
is also version tag to get the latest release of an specific version.

To get to this sort of tagging strategy some modifications are required in our
workflow and toolchain.

* Need correct versions of KIWI and skopeo in relevant streams (mostly SLE 15)
  With it the stable tags could be already defined ('latest' and '10.3' of the
  above example)
  
* Need OBS to add/modify tags at build time using the KIWI command call to
  append the build ID and/or release numbers. Currently the tag can be modified
  at call time in KIWI with the `--set-container-tag`. Probably this can be
  used to append release numbers to a tag.
  
* Need OBS to push to the registry the image with all the tags included within
  the built binary.

* SUSE requires some tool to track changes in the registry and be capable to
  detect when updates are available.
  
Tagging an image with the main package version it can be achieved by using the
`obs-service-replace_using_package_version` service. This is currently being
used in Head project in IBS & OBS.

One of the advantages of using this strategy is that it does not necessarily
require a Kubernetes manifest update for each image update. The manifest could
be set to some stable mariadb version (e.g. 10.2) and still be updated if the
image requires a rebuild due some security fixes in the SLE base image or even
in mariadb itself.

Also, using this approach, will be simple to clearly identify a local image
running in a cluster as all supported images will always be in the registry
tagged at least with a complete version and release numbers.
