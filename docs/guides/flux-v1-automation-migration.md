# Migrating image update automation to Flux v2

Image update automation is a process in which Flux makes commits to
your Git repository, when it detects that there is a new image to be
used in a workload (e.g., a Deployment). In Flux v2 this works quite
differently to how it worked in Flux v1. This guide explains the
differences and how to port your configuration from v1 to v2.

## Overview of changes between v1 and v2

In Flux v1, image update automation (from here, just "automation") was
built into the Flux daemon which scanned everything it found in the
cluster, and updated the Git repository it was syncing (applying to
the cluster).

In line with the move to custom resources in Flux v2, automation is
now done by two controllers: one which scans image repositories to
find the latest images, and one which uses that information to commit
changes to git repositories. These are in turn separate to the syncing
machinery.

The first big difference is that **automation in Flux v2 is governed
by custom resources**, whereas in Flux v1 it scanned everything, and
looked at annotations on the resources to determine what to update. In
other words, automation in v2 is more explicit than in v1 -- you have
to mention exactly which images you want to be scanned, and which
fields you want to be updated.

A consequence of using custom resources is that with Flux v2 you can
have an arbitrary number of automations, targeting different Git
repositories if you wish, and updating different sets of images. If
you run a multitenant cluster, the tenants can define automation in
their own namespaces, for their own Git repositories.

The ways in which you can choose to select an image have changed. In
Flux v1, you generally supply a filter pattern, and the latest image
is the one with the most recent build time. **In Flux v2, you choose
an ordering, and optionally a filter for the tags under
consideration**. These are dealt with in detail below.

Lastly, **in Flux v2 the fields to update in files are marked
explicitly**, rather than inferred from the type of resource along
with the annotations given. The approach in Flux v1 was limited to the
types that were programmed in, whereas Flux v2 can update any
Kubernetes object (and some files that aren't Kubernetes objects, like
`kustomization.yaml`).

To migrate to Flux v2 automation, you will need to do two things:

 - translate Flux v1 automation annotations to Flux v2 ImagePolicy
   objects, and markers in files.
 - declare the automation with an ImageUpdateAutomation object

## Preparing for migration

%%% TODO

 - when to do this is relation to migrating sync machinery
 - switching off Flux v1 automation
 - reinstating automation a bit at a time (opting in)

## Migrating a manifest to Flux v2

For each manifest (description of a Kubernetes object) that you have
put Flux v1 annotations in, you'll need to create an ImagePolicy
object, and replace the annotations with field markers. The following
sections explain these steps, and give a worked example based on the
following Deployment:

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

### How to migrate annotations to image policies

For each image that is the subject of automation -- e.g.,
`docker.io/org/my-app` in the example -- you will need to create an
`ImageRepository` object, so that the image repository is scanned. The
command-line tool helps here:

```bash
$ flux create image repository podinfo-image -n default \
    --image ghcr.io/stefanprodan/podinfo \
    --interval 5m \
    --export > podinfo-image.yaml
$ cat podinfo-image.yaml
---
apiVersion: image.toolkit.fluxcd.io/v1alpha1
kind: ImageRepository
metadata:
  name: podinfo-image
  namespace: default
spec:
  image: ghcr.io/stefanprodan/podinfo
  interval: 5m0s
```

If you are using the same image repository in several manifests, you
only need one `ImageRepository` object for it.

For each _field_ that's being updated by automation, you'll need an
`ImagePolicy` object to describe how to pick the image. In the
example, the field `.image` in the container named `"app"` is the
field being updated.

In Flux v1, annotations describe _how_ to select an image using a
prefix. In the example, the prefix is `semver:`. These are the
prefixes supported:

| Flux v1 prefix   | Meaning     |
|------------------|-------------|
| `glob:`          | Filter for tags matching the glob pattern, then select the newest by build time |
| `regex:`         | Filter for tags matching the regular expression, then select the newest by build time |
| `semver:`        | Filter for tags that represent versions, and select the highest version in the given range |

Note that ordering and filtering are fixed together -- this means you
cannot, for example, tag your images `dev-0.1.0`, `prod-1.0.0`, etc.,
then select the highest version with the prefix `dev-`.

Flux v2 does not support ordering by build time, because that requires
the image config for each image to be fetched, and this tends to fall
foul of image registry rate limiting (e.g., by
[DockerHub][dockerhub-rates]). If you are currently relying on the
build time ordering, instead start tagging your images with a
timestamp, or [CalVer version][calver], or build sequence number (CI
systems may supply a build number in the environment).

If you are using ...

|  Prefix | Flux v2 equivalent |
|---------|--------------------|
| glob:   | [Use timestamped tags](#how-to-use-timestamps-in-image-tags) |
| regex:  | [Use timestamped tags](#how-to-use-timestamp-in-image-tags) |
| semver: | [Use semver ordering](#how-to-use-semver-image-tags) |

The example uses `semver`, so the policy object follows:

```bash
$ flux create image policy my-app-policy -n default \
    --image-ref podinfo-image \
    --semver '^5.0' \
    --export > my-app-policy.yaml
$ cat my-app-policy.yaml
---
apiVersion: image.toolkit.fluxcd.io/v1alpha1
kind: ImagePolicy
metadata:
  name: my-app-policy
  namespace: default
spec:
  imageRepositoryRef:
    name: podinfo-image
  policy:
    semver:
      range: ^5.0
```

### How to use timestamps in image tags

%%% worked example

### How to use SemVer image tags

%%% worked example

## How to mark up files for update

The last thing to do in each manifest is to mark the fields that you
want to be updated.

In Flux v1, the annotations in a manifest determined the fields to be
updated. In the example:

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

... the annotations target the image used by the container `app`. This
works predictably for Deployment manifests, but when it comes to
HelmRelease manifests, it [gets complicated][helm-auto].

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
        image: ghcr.io/stefanprodan/podinfo:5.0.0 # {"$imagepolicy": "default:my-app-policy"}
```

[helm-auto]: https://docs.fluxcd.io/en/1.21.1/references/helm-operator-integration/#automated-image-detection).

## Switching automation on

%%% making an image update automation object

# %%% TODO %%%

**What to do if you run a build step which loses comments**

**How to lock / remove / pause automation**

**Is there an equivalent to the 'ignore' annotation?**

[dockerhub-rates]: https://docs.docker.com/docker-hub/billing/faq/#pull-rate-limiting-faqs
