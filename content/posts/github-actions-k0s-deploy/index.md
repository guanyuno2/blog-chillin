---
title: "Deploying a Next.js App to k0s with GitHub Actions"
date: 2026-05-11T10:00:00+07:00
draft: false
tags:
  - github-actions
  - kubernetes
  - k0s
  - nextjs
  - docker
  - traefik
categories:
  - devops
author: "Ty Van"
summary: "A practical CI/CD setup for building a Next.js app, publishing a container image, and deploying it to a k0s Kubernetes cluster through GitHub Actions."
ShowToc: true
TocOpen: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowCodeCopyButtons: true
---

I recently set up a simple deployment pipeline for a Next.js app running on a small k0s Kubernetes cluster. The goal was straightforward: push to `main`, build the app, publish a container image, and roll it out to Kubernetes automatically.

This post summarizes the setup without exposing private infrastructure details such as server IPs, SSH users, private keys, or cluster-specific credentials.

## Architecture

The deployment flow looks like this:

1. GitHub Actions checks out the repository.
2. It installs dependencies with `npm ci`.
3. It runs linting and builds the Next.js app.
4. It builds a Docker image.
5. It pushes the image to GitHub Container Registry.
6. It renders Kubernetes manifests with the correct image tag and hostname.
7. It copies the rendered manifests to the VPS over SSH.
8. It applies the manifests using `k0s kubectl`.
9. Kubernetes rolls out the new version behind Traefik.

The app itself is stateless, so it does not need a persistent volume.

## Repository Changes

The deployment setup added these files:

```text
Dockerfile
.dockerignore
.github/workflows/deploy.yml
k8s/namespace.yaml
k8s/deployment.yaml
k8s/service.yaml
k8s/ingress.yaml
public/.gitkeep
```

The Next.js config was also updated to enable standalone output:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: "standalone",
};

export default nextConfig;
```

Standalone output makes the Docker image smaller and easier to run because Next.js produces a self-contained server bundle.

## Docker Image

The Dockerfile uses a multi-stage build:

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
ENV NEXT_TELEMETRY_DISABLED=1
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

The image runs as a non-root user and serves the app on port `3000`.

## Kubernetes Resources

The Kubernetes setup uses four basic manifests:

```text
k8s/namespace.yaml
k8s/deployment.yaml
k8s/service.yaml
k8s/ingress.yaml
```

The `Deployment` runs two replicas and uses lightweight health checks:

```yaml
readinessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 10
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 30
  periodSeconds: 30
```

The `Service` exposes the app inside the cluster on port `80`, forwarding traffic to the container's `3000` port.

The `Ingress` uses Traefik:

```yaml
spec:
  ingressClassName: traefik
  rules:
    - host: APP_HOST_PLACEHOLDER
```

The hostname is replaced during the GitHub Actions run using a repository secret.

## GitHub Actions Workflow

The workflow runs on pushes to `main` and can also be triggered manually:

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

It grants only the permissions needed to read repository contents and write container packages:

```yaml
permissions:
  contents: read
  packages: write
```

The workflow then:

1. Installs dependencies.
2. Runs linting.
3. Builds the app.
4. Logs in to GHCR.
5. Builds and pushes the image.
6. Renders the Kubernetes manifests.
7. Copies the manifests to the server.
8. Applies them with `k0s kubectl`.

The image name is normalized to lowercase before pushing to GHCR:

```bash
IMAGE_NAME="$(echo "ghcr.io/${GITHUB_REPOSITORY}" | tr '[:upper:]' '[:lower:]')"
```

That avoids registry errors if the GitHub owner or repository name contains uppercase characters.

## Required Secrets

The workflow expects these GitHub repository secrets:

```text
VPS_HOST=<server-hostname-or-ip>
VPS_USER=<ssh-user>
VPS_SSH_KEY=<private-ssh-key>
VPS_SSH_PORT=<ssh-port>
APP_HOST=<public-app-hostname>
```

Do not commit these values into the repository.

The public hostname can be something like:

```text
hub.example.com
```

## Deploy Command on the Server

The workflow applies manifests through SSH using the server's k0s kubeconfig:

```bash
k0s kubectl --kubeconfig <k0s-admin-kubeconfig> apply -f /path/to/manifests
```

If the SSH user is not root, the workflow can use passwordless sudo:

```bash
sudo -n k0s kubectl --kubeconfig <k0s-admin-kubeconfig> apply -f /path/to/manifests
```

That means the deploy user needs permission to run the required `k0s kubectl` commands non-interactively.

## Verification

Before trusting the pipeline, I verified the local app checks:

```bash
npm run lint
npm run build
```

The production build created the standalone output used by the Docker image.

After deployment, useful cluster checks are:

```bash
k0s kubectl --kubeconfig <k0s-admin-kubeconfig> get pods,deploy,svc,ingress -n <namespace>
```

And the public endpoint can be checked with:

```bash
curl -I https://<public-app-hostname>
```

Expected results are a healthy deployment, ready pods, a ClusterIP service, and an Ingress route handled by Traefik.

## Rollback

Kubernetes gives a simple rollback path:

```bash
k0s kubectl --kubeconfig <k0s-admin-kubeconfig> rollout undo deployment/<app-name> -n <namespace>
k0s kubectl --kubeconfig <k0s-admin-kubeconfig> rollout status deployment/<app-name> -n <namespace> --timeout=180s
```

Because the workflow tags images with the Git commit SHA, every deployment can be traced back to a specific commit.

## Notes

If the GHCR package is private, the Kubernetes cluster needs an image pull secret. Another option is to make the package public if the image does not contain private code or secrets.

For small clusters, it is also worth checking storage before adding any workloads that need persistent volumes. A stateless app avoids that problem completely.

## Result

The final setup gives a clean deployment path:

```text
git push -> GitHub Actions -> GHCR -> VPS over SSH -> k0s Kubernetes -> Traefik -> public hostname
```

It is small, understandable, and easy to roll back.
