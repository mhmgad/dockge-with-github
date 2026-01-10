# Docker Build and Push Setup

This repository is configured to automatically build and push Docker images to `docker.io/mhmgad/dockge-plus` using GitHub Actions.

## Prerequisites

- Docker Hub account (mhmgad)
- GitHub repository with Actions enabled

## GitHub Secrets Configuration

To enable the automated Docker build and push workflow, you need to configure the following GitHub secret:

### Required Secret

**DOCKER_PASSWORD**
- Navigate to your repository settings: `Settings` → `Secrets and variables` → `Actions`
- Click `New repository secret`
- Name: `DOCKER_PASSWORD`
- Value: Your Docker Hub password or access token

#### Getting a Docker Hub Access Token (Recommended)

Using an access token is more secure than using your password:

1. Log in to [Docker Hub](https://hub.docker.com/)
2. Go to `Account Settings` → `Security` → `Access Tokens`
3. Click `New Access Token`
4. Give it a descriptive name (e.g., "GitHub Actions - dockge-plus")
5. Set permissions to `Read, Write, Delete` (or `Read & Write` minimum)
6. Copy the generated token and save it as the `DOCKER_PASSWORD` secret

## Workflow Overview

The workflow (`.github/workflows/docker-build-push.yml`) consists of three jobs:

### 1. Build Base Image (`build-base`)
- Builds the base Node.js image with Docker CLI
- Tags: `mhmgad/dockge-plus:base`
- Platforms: `linux/amd64`, `linux/arm64`, `linux/arm/v7`

### 2. Build Healthcheck Image (`build-healthcheck`)
- Builds the healthcheck binary using Go
- Tags: `mhmgad/dockge-plus:build-healthcheck`
- Platforms: `linux/amd64`, `linux/arm64`, `linux/arm/v7`

### 3. Build Main Application Image (`build-main`)
- Builds the main Dockge application
- Tags: `mhmgad/dockge-plus:latest`, `mhmgad/dockge-plus:<version>`, and more
- Platforms: `linux/amd64`, `linux/arm64`, `linux/arm/v7`
- Depends on: base and healthcheck images (waits for them to be available in Docker Hub)
- Note: Only runs for pushes to master, not for pull requests

## Initial Setup - First Build

**Important**: The first time you run the workflow, the main application build will fail because it depends on base images that don't exist yet in your Docker Hub repository. Follow these steps:

### Option 1: Two-Phase Build (Recommended)

1. **First Run - Build Base Images Only**:
   - Create and push a release tag (e.g., `v1.0.0`) to trigger the workflow
   - The base and healthcheck jobs will succeed and push images to Docker Hub
   - The main application job will likely fail (or take a long time) because it waits for base images
   - This is expected behavior for the first run

2. **Second Run - Build Everything**:
   - Once the base images are in Docker Hub, trigger the workflow again
   - You can do this by:
     - Using "Re-run workflow" in GitHub Actions
     - Using the "Run workflow" button (manual trigger)
   - This time all jobs should succeed

### Option 2: Manual Base Image Build

Alternatively, you can build and push the base images manually first:

```bash
# Clone the repository
git clone https://github.com/mhmgad/dockge-with-github.git
cd dockge-with-github

# Login to Docker Hub
docker login -u mhmgad

# Build and push base image
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t mhmgad/dockge-plus:base \
  -f ./docker/Base.Dockerfile . --push

# Build and push healthcheck image
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 \
  -t mhmgad/dockge-plus:build-healthcheck \
  -f ./docker/BuildHealthCheck.Dockerfile . --push
```

After the base images are in Docker Hub, the workflow will work normally.

## Triggering the Workflow

The workflow runs automatically on:
- **Version tags**: When you push tags like `v1.0.0`, it creates versioned images and triggers the build
- **Manual trigger**: Use the "Run workflow" button in GitHub Actions

## Docker Images Produced

After a successful workflow run, the following images will be available on Docker Hub:

- `mhmgad/dockge-plus:base` - Base image with Node.js and Docker CLI
- `mhmgad/dockge-plus:build-healthcheck` - Healthcheck binary
- `mhmgad/dockge-plus:latest` - Latest main application (from release tags)
- `mhmgad/dockge-plus:<version>` - Specific version (e.g., 1.4.2)

## Using the Docker Images

### With Docker Compose

The `compose.yaml` file in this repository is configured to use the new image:

```yaml
services:
  dockge:
    image: mhmgad/dockge-plus:latest
    # ... rest of configuration
```

Run with:
```bash
docker compose up -d
```

### Standalone Docker Run

```bash
docker run -d \
  -p 5001:5001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ./data:/app/data \
  -v /opt/stacks:/opt/stacks \
  -e DOCKGE_STACKS_DIR=/opt/stacks \
  --name dockge \
  mhmgad/dockge-plus:latest
```

## Troubleshooting

### Workflow Fails with Authentication Error

- Verify the `DOCKER_PASSWORD` secret is correctly set
- If using Docker Hub password, consider switching to an access token
- Check that the Docker Hub account `mhmgad` has permissions to push to `dockge-plus` repository

### Image Not Found When Running Compose

- Ensure the workflow has completed successfully
- Check Docker Hub to verify images were pushed: https://hub.docker.com/r/mhmgad/dockge-plus
- Try pulling manually: `docker pull mhmgad/dockge-plus:latest`

### Build Fails for Specific Platform

- Check the workflow logs in GitHub Actions
- Verify QEMU and Docker Buildx are properly configured (handled automatically in workflow)

## Notes

- The workflow uses GitHub Actions cache to speed up subsequent builds
- Pull requests will build but not push images to Docker Hub
- Multi-platform builds may take 10-20 minutes depending on the changes
