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

With the current workflow, where each container image build is pushed to a
registry, some strategy to identify versions and releases from the registry
client perspective is not as abvious as it should. 

It is important to note that container image updates happen due to a source
change (KIWI descriptor file or Dockerfile update) or due to a former package
update. Thus any update of a package required during the image build,
triggers a new image release to the registry. According to that images
publised in the registry are expected to be updated often.

Having updates on images without any visible change into the image references
can be confusing from the registry client perspective. In this context pooling
tags for a specific namespace will result always into the same list, being 
immutable across updates. 

```
GET /v2/<name>/tags/list
```

However downloaded binaries of an image will not be immutable. It should
be easy from the registry client perspective to track image updates and not
just endup with different image binaries and wihout any option to reference
an old image without having to use the image digest.

Another situation where the tags of the image should be used with care is when
the image delivers certain specific application and the image is tagged
according to this application version. This practice is helpful in
order to create readable and meaningfull image references. 

For instance, a reference like `<registry>/opensuse/mariadb:10.2` clearly
suggests the image is providing `mariadb` v10.2.

While this may look like a simple and safe way of tagging the image it could
easly lead to some confusing inconsistences when the image updates are pushed
to the registry in an automated fashion as it is the case with the Open
Build Service. In this case the tag 10.2 would be part of the image sources
and the match within the package version and tag would happen manually. This
clearly turns to be problematic in rolling release distributions where doing
this match manually is not really possible. But also could be problematic
if the exacte package version as tag is being used, as even in the scope of a
non rolling distribution the package version could be silently increased.


## Current situation

Tags from the registry perspective and client perspective are cheap and simple.
Having an image tagged multiple time with different refences causes no issues
and can be helpful to easily provide some additional context.

The main idea is to keep using stable tags to facilitate image references
in systems where no dynamic references are possible but also tag images in the
registry with additional dynamic tags, based on included packages versions or
image release numbers. This way we could have in the registry multiple images
with a meaningful tags pointing to a specific version, but also a stable tag
pointing to the latest version of an image.

Imagine there is a mariadb image tagged as below in the registry (emulating
`docker images` output):

```
REPOSITORY           TAG                    IMAGE ID
opensuse/mariadb     10.2                   1ade07d13d13
opensuse/mariadb     latest                 1ade07d13d13
opensuse/mariadb     10.2.15-3.2            1ade07d13d13
```

Note that the build ID is being used as the release number. Build ID are
the build numbers currently included in the resulting files of KIWI build
within the build service.

All three tags pointing to the same image. Then if a new update due to a new
mariadb release happens adding the new image could look like that:

```
REPOSITORY           TAG                    IMAGE ID
opensuse/mariadb     10.2                   70b5d81549ec
opensuse/mariadb     latest                 70b5d81549ec
opensuse/mariadb     10.2.15-3.2            1ade07d13d13
opensuse/mariadb     10.2.15-3.3            70b5d81549ec
```

Note that old image would be still accessible with the specific version tight
to the specific version and release. Again, if a new update happens but this
time due to a new mariadb version adding a third image into the registry could
result in:

```
REPOSITORY           TAG                    IMAGE ID
opensuse/mariadb     10.2                   70b5d81549ec
opensuse/mariadb     latest                 284549eacf84
opensuse/mariadb     10.2.15-3.2            1ade07d13d13
opensuse/mariadb     10.2.15-3.3            70b5d81549ec
opensuse/mariadb     10.3                   284549eacf84
opensuse/mariadb     10.3.1-3.4             284549eacf84
```

For that to happen it the build system requires three features:

* Support for multiple tags

  Multiple tags are possible in `kiwi` since v9.15.3 (only if 
  `skopeo >= 1.30` is also available present). The image tarballs in that case
  support multiple tags, but this is not reflected to the registry when pushed,
  only the main tag is included.
  However the Build Service has an alternative to include additional tags into
  the pushed images. Including a syntax like

  ```
  <!-- OBS-AddTag: opensuse/mariadb:10.3.1 -->
  ```
  into the XML kiwi file will make OBS to push the image with the provided
  additional tag.

* Support to include build ID within a tag

  This is currently possible using the above mentioned OBS specific syntax.
  Currently it is possible to do it with something like:

  ```
  <!-- OBS-AddTag: opensuse/mariadb:10.3.1-<RELEASE> -->
  ```

  In that case OBS will subsitute the `<RELEASE>` placeholder for the actual
  build ID.

* Support to tag according to the version of a package, or part of it.

  It can be achieved using the `replace_using_package_version` OBS service.
  See further information 
  [here](https://github.com/openSUSE/obs-service-replace_using_package_version)

A working example of this setup is currelty working in
[kubic-project/container-images](https://github.com/kubic-project/container-images)
repo and into the [OBS project](https://build.opensuse.org/project/show/devel:CaaSP:kubic-container)
that is synchronized with it.
