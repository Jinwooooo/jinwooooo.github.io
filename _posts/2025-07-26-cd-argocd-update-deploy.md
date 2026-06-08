---
title: "CD: ArgoCD — Update and Deploy"
date: 2025-07-26 00:00:00 +0900
categories: [DevOps]
tags: [cicd, github-actions, argocd, helm, gitops]
description: A GitHub Actions job that updates Helm values in a GitOps repo, then hands off to ArgoCD
image:
  path: thumbnail.webp
media_subpath: /assets/img/posts/2025-07-26-cd-argocd-update-deploy/
---

## Summary

This post covers the **deployment** stage of the pipeline — the CD (Continuous Deployment) phase that picks up where the [CI build]({% post_url 2025-06-18-ci-github-actions-image-build-and-push %}) leaves off. The CI job pushed a freshly built image to Harbor; this CD job rewrites the image tag in a GitOps repo and lets ArgoCD reconcile the cluster.

> **Helm** is Kubernetes' package manager — it bundles K8s manifests into reusable, parameterized **charts**, with values files (e.g. `values-dev.yaml`) that parameterize each environment's deployment. Keeping those values files in Git is the **Infrastructure-as-Code** half of this pipeline: a single `imageTag` change in `values-dev.yaml` is what triggers a deployment — no manual `kubectl` needed.
{: .prompt-info }

> **GitOps** = the cluster's desired state lives declaratively in Git; an agent (ArgoCD here) reconciles the running cluster to match. Updates happen via commits, not `kubectl`.
{: .prompt-info }

## CD Overview

![CD architecture — GHA workflow updates ArgoCD GitOps repo, EKS-hosted ArgoCD syncs to on-prem K8s, pulling the image from Harbor](architecture-overview.png)
_CD architecture: GHA updates the GitOps repo → ArgoCD syncs → workload cluster pulls the new image_

The stack:

- **Source Code Management** → GitHub
- **Workflow Orchestrator** (the brain) → GitHub Actions
- **Container Registry** → Harbor (self-hosted)
- **GitOps Engine** → ArgoCD (running on EKS)
- **Workload Cluster** → On-premise Kubernetes

The arrows on the diagram show the CD handoff: GitHub Actions edits `dx-infra-config` (the GitOps repo) → ArgoCD detects the change → ArgoCD applies the updated manifests; Kubernetes pulls the new image from Harbor and rolls out the new pods.

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

    # ... omitted, see the CI post for details ...

  # ─── Stage 2: ArgoCD GitOps (dx-infra-config) Helm values update ───
  update:
    name: Update
    needs: build-and-push
    runs-on: [self-hosted, arc-${{ github.ref_name }}]
    environment: ${{ github.ref_name }}

    env:
      TARGET_BRANCH: idcx-${{ github.ref_name }}
      IMAGE_TAG: ${{ needs.build-and-push.outputs.image_tag }}

    steps:
      - name: Generate GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ vars.GH_APP_ID }}
          private-key: ${{ secrets.GH_APP_PRIVATE_KEY }}
          owner: Wondermove-Inc
          repositories: dx-infra-config

      - name: Checkout dx-infra-config Repository
        uses: actions/checkout@v4
        with:
          repository: Wondermove-Inc/dx-infra-config
          ref: ${{ env.TARGET_BRANCH }}
          token: ${{ steps.app-token.outputs.token }}

      - name: Pull Latest to Avoid Conflicts
        run: |
          git pull origin ${{ env.TARGET_BRANCH }} --rebase \
            || (echo "Rebase conflict; aborting" && git rebase --abort && exit 1)

      - name: Install yq (YAML processor)
        run: |
          curl -fsSL https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -o "$RUNNER_TEMP/yq"
          chmod +x "$RUNNER_TEMP/yq"
          echo "$RUNNER_TEMP" >> "$GITHUB_PATH"

      - name: Determine helm values file
        id: values
        run: |
          HELM_VALUES="helm/idcx-was/values-${{ github.ref_name }}.yaml"
          echo "helm_values=$HELM_VALUES" >> $GITHUB_OUTPUT
          echo "Helm values file: $HELM_VALUES"

      - name: Update Rollout imageTag
        env:
          HELM_VALUES: ${{ steps.values.outputs.helm_values }}
        run: |
          echo "Updating app.imageTag to ${IMAGE_TAG}..."
          yq eval ".app.imageTag = \"${IMAGE_TAG}\"" -i $HELM_VALUES
          echo "Updated $HELM_VALUES"

      - name: Commit & Push
        run: |
          git config user.email "ci-bot@github.com"
          git config user.name  "GitHub Actions"
          git commit -am "[idcx-was] Deploy ${IMAGE_TAG}" \
            || echo "No changes to commit"
          git push origin ${{ env.TARGET_BRANCH }}

          echo ""
          echo "Pushed update to dx-infra-config"
          echo "ArgoCD will detect the change and sync the cluster"
          echo ""
```
{: file=".github/workflows/build-push-deploy.yaml" }

> Values fetched via `vars.*` and `secrets.*` live in the GitHub Actions secrets and variables tab: **project repository → Settings → Secrets and variables → Actions**. `vars` are plain-text, `secrets` are encrypted at rest and masked in logs.
{: .prompt-tip }

![GitHub repo Settings → Secrets and variables → Actions tab](github-secrets-variables-settings.png)
_Where `vars.*` and `secrets.*` are configured_

A few framing keys on the yaml above:

- **`needs: build-and-push`** — this `update` job only runs if the CI `build-and-push` job succeeded, so a broken build never triggers a deployment.
- **`IMAGE_TAG: ${{ needs.build-and-push.outputs.image_tag }}`** — carries the short commit SHA from the CI job forward, so the tag written into Helm values is exactly the image that was just pushed to Harbor.
- **`TARGET_BRANCH: idcx-${{ github.ref_name }}`** — commits to a per-environment branch on `dx-infra-config` (the GitOps repo), so each environment's deployment state is isolated.
- **GitHub App token (not PAT)** — the `Generate GitHub App token` step issues a short-lived token scoped to a single repo (`dx-infra-config`). This is the right pattern when one repo's workflow needs to write to another repo: less blast radius than a personal access token, and not tied to any single user account whose departure would invalidate it.

End-to-end the CD process is:

1. **Retrieve a GitHub Repository Access Token** — `actions/create-github-app-token` mints a short-lived token scoped to `dx-infra-config`.
2. **Update Helm values** — checkout `dx-infra-config`, install `yq`, rewrite `app.imageTag` in the per-environment values file.
3. **Commit & push to the GitOps repo** — the bot pushes the values change to the target branch on `dx-infra-config`.
4. **ArgoCD syncs the workload cluster** — outside of GitHub Actions: ArgoCD detects the new values and reconciles the cluster.

## In Action

Here's the workflow running end-to-end. The demo covers the full CI/CD pipeline — GitHub Actions, ArgoCD, ARC, Harbor all wired up — though the CI half is covered in a separate post.

{% include embed/youtube.html id='_i946F7uWUY?start=202' %}
_Live demo: the full CI/CD pipeline_

Here's a sample run, triggered by merging a PR to `dev`:

![Update job step list with per-step durations](actions-update-job-overview.png)
_All steps in the Update job_

The screenshots below walk through the GitHub Actions side of the four steps, in the order they execute:

![Generate GitHub App token step log showing app-id, owner, and the resulting scoped token](actions-generate-github-app-token.png)
_Step 1 — Generate a short-lived token scoped to `dx-infra-config`_

![Update Rollout imageTag step log showing yq updating helm/idcx-was/values-dev.yaml](actions-update-rollout-imagetag.png)
_Step 2 — `yq` rewrites `app.imageTag` in the per-environment values file_

![Commit & Push step log showing the bot's commit and push to dx-infra-config](actions-commit-push-gitops.png)
_Step 3 — The bot commits the values change and pushes to the target branch_

**Step 4 — ArgoCD takes over.** Once the commit lands in `dx-infra-config`, GitHub Actions is done. ArgoCD is configured to watch that repo; it detects the new `imageTag`, marks the Application out-of-sync, and reconciles the cluster against the updated values.

## Closing the Loop

That closes the loop: a merge to `dev` triggers CI → CI builds and pushes an image to Harbor → CD edits Helm values in `dx-infra-config` → ArgoCD reconciles the cluster. From a developer's perspective, the only action was the merge.
