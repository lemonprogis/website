---
title: GitHub Actions Basic App Builder
linkTitle: GitHub Actions Basic App Builder
description: "How to build and push image tags for Flux from Git branches and tags."
weight: 39
---

This guide shows how to configure GitHub Actions to build an image for each new commit pushed on a branch, for PRs, or for tags in the most basic way that Flux's automation can work with and making some considerations for both dev and production.

In the [GitHub Actions Manifest Generation] guide, we saw some workflows that render manifests in CI and commit them back to Git. From [Image Update Guide] we saw that Flux can set `.spec.git.push.branch` to [Push updates to a different branch] than the one used for checkout. Those advanced techniques are above and beyond what is needed to work with Flux.

In this guide, a single GitHub Actions workflow is presented with a few variations but one simple theme: Flux's only firm requirement for integrating with CI is for the CI to build and push an image.

### Scope of this document

Strictly speaking Flux considers CI to be out-of-scope, but this answer frequently leads to bad experiences caused by over-complicated CI. This example is intended to cover a majority of use cases with the simplest possible workflow.

We anticipate in this guide that Flux users who are developing one or more apps want two build strategies in isolation for each app: a **Dev** build generates a tag with the branch name, commit hash, and timestamp; and a **Release** build produces a semantic version tag from the release tag.

How to configure an `ImageUpdateAutomation` resource to take advantage of Release or Dev builds with automation is covered separately in the [Image Update Guide] and [Sortable image tags] guide, respectively.

## Example GitHub Actions Workflow

tl;dr: This build workflow does everything that Flux needs. Drop it into `.github/workflows/docker-build.yml` and reap the benefits.

First copy this example and update `IMAGE` to point to your own image repository target. Then set `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` and you are done. Most git push events will now result in images suitable for Flux to deploy.

For a deeper understanding and some variations, see the remainder of the doc.

```yaml
name: Docker Build, Push

on:
  push:
    branches:
      - '*'
    tags-ignore:
      - 'release/*'

jobs:
  docker:
    env:
      IMAGE: kingdonb/any_old_app
    runs-on: ubuntu-latest
    steps:
      - name: Prepare
        id: prep
        run: |
          BRANCH=${GITHUB_REF##*/}
          TS=$(date +%s)
          REVISION=${GITHUB_SHA::8}
          BUILD_ID="${BRANCH}-${REVISION}-${TS}"
          LATEST_ID=canary
          if [[ $GITHUB_REF == refs/tags/* ]]; then
            BUILD_ID=${GITHUB_REF/refs\/tags\//}
            LATEST_ID=latest
          fi
          echo ::set-output name=BUILD_DATE::$(date -u +'%Y-%m-%dT%H:%M:%SZ')
          echo ::set-output name=BUILD_ID::${BUILD_ID}
          echo ::set-output name=LATEST_ID::${LATEST_ID}

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v1

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          push: true
          tags: |
            ${{ env.IMAGE }}:${{ steps.prep.outputs.BUILD_ID }}
            ${{ env.IMAGE }}:${{ steps.prep.outputs.LATEST_ID }}

      - name: Image digest
        run: echo ${{ steps.docker_build.outputs.digest }}
```

This workflow incorporates a few key concepts and properties which are important for Flux.

### Workflow Event Triggers

There are two paths through this flow: when a commit is pushed to any branch and when a commit is pushed to any tag, with some exceptions possible as shown with `tags-ignore:` – this example is given in case you are using the `release/*` tags as shown in the [Jsonnet Render Action] example.

These workflows are executed by GitHub Actions on the `push` event for any branches and tags we specify.

```yaml
on:
  push:
    branches:
      - '*'
    tags-ignore:
      - 'release/*'
```

You may want to invert or adjust these patterns depending on how you are using branches, image tags, and git tags. Flux is not prescriptive about any of this. Maybe you only build tags that match a certain pattern, or only commits on the `main` branch, depending on the need. Some variations are expected and they are out of scope for this guide.

Now let's walk through the rest of this example workflow.

### Docker Build job

The workflow has one job with the id `docker` whose purpose is to turn commits from push events into deployable images.

An individual image tag name (string) has two parts, `IMAGE` which represents the image name that is common for all images in the same project, and following that image name separated by a colon is a `tag` which uniquely identifies a revision of the image. Repositories can hold many tags, and tags can utilize various forms and formats.

#### Mutable vs. Immutable tags

Image tags can be mutable or immutable. Flux works best with immutable tags: `latest` and `canary` are examples of mutable tags.

This example produces both mutable and immutable tags because Flux works with immutable tags, but many users still expect a `latest` tag even if Flux won't use it. Mutable tags are useful for example with environment branches, to stably represent the latest build in a named environment, but they are contrary to GitOps, and Flux automation demands a timestamp or something sortable in the tag string. Thus mutable tags alone are not suitable for any Flux purpose.

In this example, `LATEST_ID` represents a mutable tag and `latest` as a tag represents the last release build that was pushed from any Git tag. The `canary` tag is the last image that was pushed from any branch.

`BUILD_ID` represents the immutable tag in both the dev and release path. This is either a literal tag string from Git tag (Flux works best with semver tags) or a `${BRANCH}-${REVISION}-${TS}` in this build workflow.

The mutable tags `canary` and `latest` are chosen by the script depending on which event triggered the build. If the image is built from a tag, the `latest` tag is used. If it is built from a branch, `canary` is used instead. These tags will therefore always point at the "latest" release tag and the latest "canary" however you define it.

This example shows one useful convention among many possible uses for mutable image tags.

Another sensible choice could be to build and push canary images only from the `main` branch. This script can be as elaborate as you want, the important logic is all contained in the shell script embedded in the `Prepare` step:

### Prepare Step

```yaml
  steps:
  - name: Prepare
    id: prep
    run: |
      BRANCH=${GITHUB_REF##*/}
      TS=$(date +%s)
      REVISION=${GITHUB_SHA::8}
      BUILD_ID="${BRANCH}-${REVISION}-${TS}"
      LATEST_ID=canary
      if [[ $GITHUB_REF == refs/tags/* ]]; then
        BUILD_ID=${GITHUB_REF/refs\/tags\//}
        LATEST_ID=latest
      fi
      echo ::set-output name=BUILD_DATE::$(date -u +'%Y-%m-%dT%H:%M:%SZ')
      echo ::set-output name=BUILD_ID::${BUILD_ID}
      echo ::set-output name=LATEST_ID::${LATEST_ID}
```

This script has no external effects, it only takes some inputs from environment variables set by GitHub Actions and calculates them into several outputs: `BUILD_ID` and `LATEST_ID`. The `BUILD_DATE` is also exported as an output for informational purposes and is not used elsewhere in the workflow.

`TS` is the Unix timestamp in seconds, a monotonically increasing value that represents when the build got scheduled. This lets us reliably determine what build is actually latest, even when some builds may take longer or shorter.

{{% alert title="Use Immutable Tags" %}}
This section highlights another advantage of Flux's requirement for using timestamped tags instead of a mutable `latest` tag, in which case the longest build (and not necessarily the latest promoted build) can occasionally win out.
{{% /alert %}}

`REVISION` is the first 8 characters of the `GITHUB_SHA`, a fingerprint that is kept for humans to differentiate more easily between tags strings that are very similar. It is not meaningful for Flux and can be omitted if preferred. Only `TIMESTAMP` has any function as it is needed to create an `ImagePolicy` (reference: [Sortable image tags]).

### Dependencies Setup

These steps prepare the build environment with QEMU and Docker:

```yaml
      - name: Set up QEMU
# ...
      - name: Set up Docker Buildx
```

### DockerHub Login

Secrets for your container registry with read and write access can be added in GitHub as [Encrypted secrets] and retrieved for use when pushing images.

```yaml
      - name: Login to DockerHub
# ...
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
```

If the GitHub Container Registry ([GHCR.io][Working with GHCR]) is used, users can skip encrypting secrets and use the `write:packages` scope with ambient `GITHUB_TOKEN` instead. The [Docker Login action] has more specific instructions.

### Build and push tag(s)

Now that Docker is logged in, a generic build and push is invoked, pushing both a mutable and an immutable image tag:

```yaml
      - name: Build and push
        id: docker_build
# ...
        with:
          push: true
          tags: |
            ${{ env.IMAGE }}:${{ steps.prep.outputs.BUILD_ID }}
            ${{ env.IMAGE }}:${{ steps.prep.outputs.LATEST_ID }}
```

An image digest is printed at the end for information.

```yaml
      - name: Image digest
        run: echo ${{ steps.docker_build.outputs.digest }}
```

#### Image Provenance Security

Note this guide does not cover or implement Image Provenance or any cryptographic signature, but Flux itself does provide those examples as they are implemented within its own controllers!

Another exercise for the reader to improve on this example builder could be implementing Cosign for cryptographically proving the image provenance as described in [Security: Image Provenance].

[GitHub Actions Manifest Generation]: /docs/use-cases/gh-actions-manifest-generation/]
[Push updates to a different branch]: /docs/guides/image-update/#push-updates-to-a-different-branch
[Image Update Guide]: /docs/guides/image-update/
[Sortable image tags]: /docs/guides/sortable-image-tags/
[Jsonnet Render Action]: /docs/use-cases/gh-actions-manifest-generation/#jsonnet-render-action
[Working with GHCR]: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
[Docker Login action]: https://github.com/docker/login-action#github-container-registry
[Security: Image Provenance]: /blog/2022/02/security-image-provenance/
