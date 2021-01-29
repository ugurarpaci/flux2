# Migrating image update automation to Flux v2

"Image Update Automation" is a process in which Flux makes commits to
your Git repository when it detects that there is a new image to be
used in a workload (e.g., a Deployment). In Flux v2 this works quite
differently to how it worked in Flux v1. This guide explains the
differences and how to port your configuration from v1 to v2.

## Overview of changes between v1 and v2

In Flux v1, image update automation (from here, just "automation") was
built into the Flux daemon which scanned everything it found in the
cluster, and updated the Git repository it was syncing (applying to
the cluster).

In Flux v2,

 - automation is controlled with custom resources, not annotations
 - ordering images by build time is not supported (see
   [below](#how-to-use-timestamp-in-image-tags) for what to do
   instead)
 - the fields to update are marked explicitly, rather than inferred
   from annotations.

**Automation is now controlled by custom resources**

Flux v2 breaks the functions in Flux v1's daemon down into
controllers, with each having a specific area of concern. Automation
is now done by two controllers: one which scans image repositories to
find the latest images, and one which uses that information to commit
changes to git repositories. These are in turn separate to the syncing
machinery.

This means that automation in Flux v2 is governed by custom
resources. In Flux v1 the daemon scanned everything, and looked at
annotations on the resources to determine what to update. Automation
in v2 is more explicit than in v1 -- you have to mention exactly which
images you want to be scanned, and which fields you want to be
updated.

A consequence of using custom resources is that with Flux v2 you can
have an arbitrary number of automations, targeting different Git
repositories if you wish, and updating different sets of images. If
you run a multitenant cluster, the tenants can define automation in
their own namespaces, for their own Git repositories.

**Selecting an image is more flexible**

The ways in which you choose to select an image have changed. In Flux
v1, you generally supply a filter pattern, and the latest image is the
image with the most recent build time out of those filtered. In Flux
v2, you choose an ordering, and separately specify a filter for the
tags to consider. These are dealt with in detail below.

Selecting an image by build time is no longer supported. This is the
implicit default in Flux v1. In Flux v2, you will need to tag images
so that they sort in the order you would like -- [see
below](#how-to-use-timestamps-in-image-tags) for how to do this
conveniently.

**Fields to update are explicitly marked**

Lastly, in Flux v2 the fields to update in files are marked
explicitly. In Flux v1 they are inferred from the type of the
resource, along with the annotations given. The approach in Flux v1
was limited to the types that had been programmed in, whereas Flux v2
can update any Kubernetes object (and some files that aren't
Kubernetes objects, like `kustomization.yaml`).

## Preparing for migration

It is best to complete migration of your system to _Flux v2 syncing_
first, using the [Flux v1 migration guide][flux-v1-migration]. This
will remove Flux v1 from the system, along with its image
automation. You can then reintroduce automation with Flux v2 by
following the instructions in this guide.

It is safe to leave the annotations for Flux v1 in files while you
reintroduce automation, because Flux v2 will ignore them.

To migrate to Flux v2 automation, you will need to do three things:

 - make sure you are running the automation controllers; and,
 - translate Flux v1 annotations to Flux v2 `ImageRepository` and
   `ImagePolicy` objects, and put update markers in files; and,
 - declare the automation with an `ImageUpdateAutomation` object

The automation objects mentioned above do _not_ have to be in the same
namespace as the objects in files to be updated by automation. In
fact, the examples here will put all automation objects in the
`flux-system` namespace, to keep them separate from applications
running in the cluster.

## Running the automation controllers

**After migrating Flux v1 _read-only_**

If you followed the [Flux v1 migration guide][flux-v1-migration], you
will already be running some Flux v2 controllers. The automation
controllers are currently considered an optional extra to those, but
are installed and run in much the same way:

```bash
$ flux install \
    --components=image-reflector-controller,image-automation-controller
```

It is safe to repeat the installation command, or to run it after
using `flux bootstrap`.

!!! hint
    Use `--export` to print the YAML to `stdout`, if you want to
    commit it to git before applying.

**With a fresh cluster and Git repository**

When starting from scratch, you will usually use `flux
bootstrap`. Include the image automation controllers in your starting
configuration with the flag `--components-extra`, [as shown in the
installation guide][flux-bootstrap].

## Migrating each manifest to Flux v2

In Flux v1, the annotation

    fluxcd.io/automated: "true"

switch automation on for a manifest (description of a Kubernetes
object). For each manifest that has that annotation, you will need to
create custom resources to scan for the latest image, and to replace
the annotations with field markers.

The following sections explain these steps, using this example
Deployment manifest which is initially annotated to work with Flux v1:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
  annotations:
    fluxcd.io/automated: "true"
    fluxcd.io/tag.app: semver:^5.0
spec:
  template:
    spec:
      containers:
      - name: app
        image: ghcr.io/stefanprodan/podinfo:5.0.0
```

!!! warning
    A YAML file may have more than one manifest in it, separated with
    `---`. Be careful to account for each manifest in a file.

### How to migrate annotations to image policies

For each image repository that is the subject of automation -- e.g.,
`ghcr.io/stefanprodan/podinfo` in the example -- you will need to
create an `ImageRepository` object, so that the image repository is
scanned.

The command-line tool [`flux`][install-cli] helps here:

```bash
$ flux create image repository podinfo-image \
    --image ghcr.io/stefanprodan/podinfo \
    --interval 5m \
    --export > podinfo-image.yaml
$ cat podinfo-image.yaml
---
apiVersion: image.toolkit.fluxcd.io/v1alpha1
kind: ImageRepository
metadata:
  name: podinfo-image
  namespace: flux-system
spec:
  image: ghcr.io/stefanprodan/podinfo
  interval: 5m0s
```

!!! hint
  If you are using the same image repository in several
  manifests, you only need one `ImageRepository` object for it.

For each _field_ that's being updated by automation, you'll need an
`ImagePolicy` object to describe how to select an image for the field
value. In the example, the field `.image` in the container named
`"app"` is the field being updated.

In Flux v1, annotations describe how to select the image to update to,
using a prefix. In the example, the prefix is `semver:`:

```yaml
  annotations:
    fluxcd.io/automated: "true"
    fluxcd.io/tag.app: semver:^5.0
```

These are the prefixes supported in Flux v1:

| Flux v1 prefix   | Meaning     |
|------------------|-------------|
| `glob:`          | Filter for tags matching the glob pattern, then select the newest by build time |
| `regex:`         | Filter for tags matching the regular expression, then select the newest by build time |
| `semver:`        | Filter for tags that represent versions, and select the highest version in the given range |

Flux v2 does not support selecting the lastest image by build time,
because that requires the container config file for each image to be
fetched, and this operation is subject to strict rate limiting by
image registries (e.g., by [DockerHub][dockerhub-rates]). The
suggested way to select by image build time is to put a timestamp in
each image tag, as explained below.

If you are using ...

|  Flux v1 prefix | Flux v2 equivalent |
|-----------------|--------------------|
| glob:           | [Use timestamped tags](#how-to-use-timestamps-in-image-tags) |
| regex:          | [Use timestamped tags](#how-to-use-timestamp-in-image-tags) |
| semver:         | [Use semver ordering](#how-to-use-semver-image-tags) |

### How to use timestamps in image tags

To give image tags a useful ordering, you can use a timestamp as part
of each image's tag, then sort alphabetically.

This is a change from Flux v1, in which the build time was fetched
from each image's config, and didn't need to be included in the image
tag. Therefore, this is likely to require a change to your build
process.

**Example of a build process with timestamp tagging**

Here is an example of a [GitHub Actions job][gha-syntax] that creates
a "build ID" with the git branch, SHA1, and a timestamp, and uses it
as a tag when building an image:

```yaml
jobs:
  build-push:
    env:
        IMAGE: org/my-app
    runs-on: ubuntu-latest
    steps:

    - name: Generate build ID
      id: prep
      run: |
          branch=${GITHUB_REF##*/}
          sha=${GITHUB_SHA::8}
          ts=$(date +%s)
          echo "::set-output name=BUILD_ID::${branch}-${sha}-${ts}"

    # These are prerequisites for the docker build step
    - name: Set up QEMU
      uses: docker/setup-qemu-action@v1
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v1
    - name: Login to DockerHub
      uses: docker/login-action@v1
      with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

    - name: Publish multi-arch container image
      uses: docker/build-push-action@v2
      with:
          push: true
          context: .
          file: ./Dockerfile
          tags: |
            ${{ env.IMAGE }}:${{ steps.prep.outputs.BUILD_ID }}
```

**Formats and alternatives**

The important properties for sorting alphabetically to work well are
that the parts of the timestamp go from most significant to least
(e.g., the year down to the second), and that the output is always the
same number of characters.

Image tags often show in user interfaces, so readability matters. Here
are some alternatives:

```bash
$ # seconds-since-epoch (used in the example above)
$ date +%s
1611840548
$ # date and time (remember ':' is not allowed in a tag)
$ date +%F.%H%M%S
2021-01-28.133158
```

Alternatively, you can use a stable serial number as part of the tag.
Some CI platforms will provide a build number in an environment
variable, but that may not be reliable to use as a serial number --
check the platform documentation.

A commit count can be a reasonable stand-in for a serial number, if
you build an image per commit, and you don't rewrite the branch in
question.

```bash
$ # commits in branch
$ git --rev-list --count HEAD
1504
```

Beware: this will not give a useful number if you have a shallow
clone.

**Filtering the tags in an `ImagePolicy`**

The timestamp (or serial number) is the part of the tag that you want
to order on. The SHA1 is there so you can track an image back to the
commit from which it was built. You don't need the branch for sorting,
but you may want to include only builds from a specific branch.

Say you want to filter for only images that are from `main` branch,
and pick the most recent. Your `ImagePolicy` would look like this:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1alpha1
kind: ImagePolicy
metadata:
  name: my-app-policy
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: podinfo-image
  filterTags:
    pattern: '^main-[a-f0-9]+-(?P<ts>[0-9]+)'
    extract: '$ts'
  policy:
    alphabetical:
      order: asc
```

The `.spec.pattern` field gives a regular expression that a tag must
match to be included. The `.spec.extract` field gives a replacement
pattern that can refer back to capture groups in the filter
pattern. The extracted values are sorted to find the selected image
tag. In this case, the timestamp part of the tag will be extracted and
sorted alphabetically in ascending order. See [the reference
docs][imagepolicy-ref] for more examples.

### How to use SemVer image tags

The other kind of sorting is by [SemVer][semver], picking the highest
version from among those included by the filter. A semver range will
also filter for tags that fit in the range. For example,

```yaml
    semver:
      range: ^5.0
```

includes only tags that have a major version of `5`, and selects
whichever is the highest.

This can be combined with a regular expression pattern, to filter on
other parts of the tags. For example, you might put a target
environment as well as the version in your image tags, like
`dev-v1.0.3`.

Then you would use an `ImagePolicy` similar to this one:

```yaml
apiVersion: image.toolkit.fluxcd.io/v1alpha1
kind: ImagePolicy
metadata:
  name: my-app-policy
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: podinfo-image
  filterTags:
    pattern: '^dev-v(?P<version>.*)'
    extract: '$version'
  policy:
    semver:
      range: '^1.0'
```

### An ImagePolicy for the example

The example Deployment has annotations using `semver:` as a prefix, so
the policy object also uses semver:

```bash
$ flux create image policy my-app-policy \
    --image-ref podinfo-image \
    --semver '^5.0' \
    --export > my-app-policy.yaml
$ cat my-app-policy.yaml
---
apiVersion: image.toolkit.fluxcd.io/v1alpha1
kind: ImagePolicy
metadata:
  name: my-app-policy
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: podinfo-image
  policy:
    semver:
      range: ^5.0
```

## How to mark up files for update

The last thing to do in each manifest is to mark the fields that you
want to be updated.

In Flux v1, the annotations in a manifest determines the fields to be
updated. In the example, the annotations target the image used by the
container `app`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
  annotations:
    fluxcd.io/automated: "true"
    fluxcd.io/tag.app: semver:^5.0 # <-- `.app` here
spec:
  template:
    spec:
      containers:
      - name: app                  # <-- targets `app` here
        image: ghcr.io/stefanprodan/podinfo:5.0.0
```

This works straight-forwardly for Deployment manifests, but when it
comes to `HelmRelease` manifests, it [gets complicated][helm-auto],
and it doesn't work at all for many kinds of resources.

For Flux v2, you mark the field you want to be updated directly, with
the namespaced name of the image policy to apply. This is the example
Deployment, marked up for Flux v2:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    namespace: default
    name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: ghcr.io/stefanprodan/podinfo:5.0.0 # {"$imagepolicy": "flux-system:my-app-policy"}
```

The value `flux-system:my-app-policy` names the policy that selects
the desired image.

This works in the same way for `DaemonSet` and `CronJob`
manifests. For `HelmRelease` manifests, put the marker alongside the
part of the `values` that has the image tag. If the image tag is a
separate field, you can put `:tag` on the end of the name, to replace
the value with just the selected image's tag. The [image automation
guide][image-auto-guide] has examples for `HelmRelease` and other
custom resources.

## Making automation go

In Flux v1, automation was run by default. With Flux v2, you have to
explicitly tell the controller which Git repository to update and how
to do so. These are defined in an `ImageUpdateAutomation` object; but
first, you need a `GitRepository` with write access, for the
automation to use.

If you followed the [Flux v1 read-only migration
guide][flux-v1-migration], you will have a `GitRepository` defined in
the namespace `flux-system`, for syncing to use. This `GitRepository`
will have read access to the Git repository by default, and automation
needs _write_ access to push commits.

%%% What to do, what to do (https://github.com/fluxcd/flux2/issues/826) %%%

# %%% TODO %%%

**What to look at to see if it's working**

**Where to put repositories and policies**

**What to do if you run a build step which loses comments**

**How to lock / remove / pause automation**

**Is there an equivalent to the 'ignore' annotation?**

[dockerhub-rates]: https://docs.docker.com/docker-hub/billing/faq/#pull-rate-limiting-faqs
[gha-syntax]: https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions
[imagepolicy-ref]: https://toolkit.fluxcd.io/components/image/imagepolicies/
[helm-auto]: https://docs.fluxcd.io/en/1.21.1/references/helm-operator-integration/#automated-image-detection).
[image-auto-guide]: https://toolkit.fluxcd.io/guides/image-update/#configure-image-update-for-custom-resources
[flux-v1-migration]: ../flux-v1-migration/
[install-cli]: https://toolkit.fluxcd.io/get-started/#install-the-flux-cli
[image-auto-guide]: ../image-update/
[flux-bootstrap]: https://toolkit.fluxcd.io/guides/installation/#bootstrap
