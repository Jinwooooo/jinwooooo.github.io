---
title: "CI: GitHub Actions — Image Build and Push"
date: 2025-06-18 00:00:00 +0900
categories: [DevOps]
tags: [cicd, github-actions, docker, harbor, arc, kubernetes]
description: A GitHub Actions workflow that builds and pushes container images to a self-hosted Harbor registry on ARC runners
image:
  path: thumbnail.webp
media_subpath: /assets/img/posts/2025-06-18-ci-github-actions-image-build-and-push/
---

## Summary

This post covers the **image build and push** part of the CI (Continuous Integration) phase — taking source code from a merged PR, building a container image, and pushing it to a registry that downstream CD can pull from.

> The CI phase in CI/CD covers tests, image build, and image push to a designated location (typically a container registry, or CR). Tests usually run on pull or merge requests; the image build and push run after a successful merge.
{: .prompt-info }

## CI Overview

> Testing is a separate process and will be covered in its own post.
{: .prompt-info }

![CI architecture overview — GitHub → ARC runner → Harbor → on-prem K8s, with EKS-hosted ArgoCD on the side](architecture-overview.png)
_CI architecture for `development` projects, mostly on-premise resources_

The stack:

- **Source Code Management** → GitHub
- **Workflow Orchestrator** (the brain) → GitHub Actions
- **Compute Layer** (the muscle) → ARC (Actions Runner Controller)
- **Container Registry** → Harbor (self-hosted)

The workflow files live in the project repository at `<root>/.github/workflows/`.

## Under the Hood

```yaml
name: Docker Image Build and Push & ArgoCD Helm Update

on:
  push:
    branches: [dev]

permissions:
  id-token: write
  contents: read

concurrency:
  group: build-push-deploy-${{ github.ref_name }}
  cancel-in-progress: true

jobs:
  # ─── Stage 1: Docker Image Build and Push to designated Registry ───
  build-and-push:
    name: Build and Push
    runs-on: [self-hosted, arc-${{ github.ref_name }}]
    environment: ${{ github.ref_name }}

    outputs:
      image_tag: ${{ steps.vars.outputs.image_tag }}

    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v4

      - name: Set Variables
        id: vars
        run: |
          IMAGE_TAG=$(git rev-parse --short HEAD)
          echo "REGISTRY_DIR=idcx-${{ github.ref_name }}/was" >> $GITHUB_ENV
          echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_ENV
          echo "image_tag=$IMAGE_TAG" >> $GITHUB_OUTPUT

      # ─── Harbor Login (dev) ───
      - name: Login to Harbor
        if: github.ref_name == 'dev'
        uses: docker/login-action@v3
        with:
          registry: ${{ vars.HARBOR_REGISTRY }}
          username: ${{ vars.HARBOR_USERNAME }}
          password: ${{ secrets.HARBOR_PASSWORD }}

      # ─── Build and Push ───
      - name: Docker Image Build and Push
        run: |
          REGISTRY="${{ vars.HARBOR_REGISTRY }}"

          docker buildx build \
            --platform linux/amd64 \
            --cache-from=type=registry,ref=$REGISTRY/${{ env.REGISTRY_DIR }}:latest \
            --cache-to=type=inline \
            -f Dockerfile \
            -t $REGISTRY/${{ env.REGISTRY_DIR }}:${{ env.IMAGE_TAG }} \
            -t $REGISTRY/${{ env.REGISTRY_DIR }}:latest \
            --push .

          echo "Pushed to $REGISTRY/${{ env.REGISTRY_DIR }}:${{ env.IMAGE_TAG }}"

  # ─── Stage 2: ArgoCD Helm Update — omitted (covered in a separate post) ───
```
{: file=".github/workflows/build-push-deploy.yaml" }

> Values fetched via `vars.*` and `secrets.*` live in the GitHub Actions secrets and variables tab: **project repository → Settings → Secrets and variables → Actions**. `vars` are plain-text, `secrets` are encrypted at rest and masked in logs.
{: .prompt-tip }

![GitHub repo Settings → Secrets and variables → Actions tab](github-secrets-variables-settings.png)
_Where `vars.*` and `secrets.*` are configured_

A few framing keys on the yaml above:

- **`concurrency`** ensures only one build runs per branch — a newer push cancels the in-flight one.
- **`environment: ${{ github.ref_name }}`** binds the job to a GitHub *environment* of the same name (`dev`, `stage`, `prod`), where environment-scoped `vars` / `secrets` live and where approval gates would attach.
- **`outputs.image_tag`** is consumed by a downstream `Update` job — the ArgoCD Helm bump, covered in a separate post.

End-to-end the CI process is:

1. **Fetch the muscle** — GitHub Actions claims an idle ARC runner.
2. **Log in to the container registry** — authenticate the runner against Harbor with a robot account.
3. **Build the image with Docker** — `docker buildx build` against the project Dockerfile, tagged with both the short commit SHA and `latest`.
4. **Push to the container registry** — the `--push` flag uploads both tags to Harbor in the same build step.

## In Action

Here's a sample run, triggered by merging a PR to `dev`:

![GitHub Actions run summary for the merged PR](actions-job-summary.png)
_Workflow run summary: Build and Push (37s) + Update (16s)_

![Build and Push step list with per-step durations](actions-build-push-steps.png)
_The full step list of the Build and Push job_

The screenshots below walk through the four steps end-to-end, in the order they execute in the workflow:

![Set up job log showing ARC runner name and version](actions-setup-job-fetch-runner.png)
_Step 1 — ARC runner picks up the job ("the muscle")_

![Login to Harbor step log](actions-login-harbor.png)
_Step 2 — Logging in to Harbor_

![Docker build step log showing buildx pulling base image and Dockerfile layers](actions-docker-build.png)
_Step 3 — Building the image with Docker_

![Docker push step log showing image pushed to harbor.example/idcx-dev/was:&lt;sha&gt;](actions-docker-push.png)
_Step 4 — Pushing to the container registry_

The image is now in Harbor, tagged with both the commit SHA and `latest`. The **CD** half of the pipeline (ArgoCD picking up the new tag via a Helm values update) is covered in a separate post.
