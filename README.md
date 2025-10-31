# devops-engineering-ci-public-build-platform-specialised-image-workflow

Reusable GitHub Actions workflow to build IQGeo Platform 7.x specialized images with multi-architecture support.

## Overview

This workflow builds 5 specialized platform images for both arm64 and amd64 architectures, creating multi-arch manifests for each. It implements a specific dependency chain to ensure images are built in the correct order.

## Workflow: `build-specialised-images.yml`

### Purpose

Builds specialized platform images that extend the base platform with specific capabilities:

1. **platform-base** - Core platform with IQGeo files extracted
2. **platform-build** - Build tools (Node.js, pip, Python build dependencies)
3. **platform-appserver** - Application server (Apache, mod_wsgi)
4. **platform-tools** - Additional tools (JDK)
5. **platform-devenv** - Development environment (debugpy, playwright, dev tools)

### Build Order & Dependencies

The workflow enforces the following dependency chain:

```
platform-base (no dependencies)
    ↓
platform-build (depends on platform-base)
    ↓
    ├─→ platform-appserver (parallel)
    └─→ platform-tools (parallel)
        ↓
    platform-devenv (depends on platform-appserver)
```

### Architecture

For each image type, the workflow:
1. Builds **arm64** version using `arm64` runner
2. Builds **amd64** version using `x64` runner (in parallel with arm64)
3. Creates **multi-arch manifest** combining both architectures
4. Pushes to both Azure Container Registry (ACR) and Harbor

This results in **15 total jobs** (5 images × 3 steps each).

## Usage

This is a reusable workflow called by the main platform build workflow:

```yaml
jobs:
  build-specialised-platform-images:
    uses: IQGeo/devops-engineering-ci-public-build-platform-specialised-image-workflow/.github/workflows/build-specialised-images.yml@main
    with:
      version: '7.4.0'
      hyphenated_module: 'dev-db'
      updated_tags: 'pre-release,latest'
      build_id: 'unique-build-id-123'
      is_release: 'false'
      engineering_prefix: 'devops_sandbox_engineering'
      releases_prefix: 'devops_sandbox_releases'
      dev_tools_version: '7.4.0'
    secrets: inherit
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `version` | Platform version to build (e.g., 7.4.0) | Yes | - |
| `hyphenated_module` | Hyphenated module name for image naming | Yes | - |
| `updated_tags` | Comma-separated list of tags (e.g., pre-release, latest) | No | - |
| `build_id` | Unique build ID per workflow run | Yes | - |
| `engineering_prefix` | Engineering prefix for Harbor registry | No | `devops_sandbox_engineering` |
| `releases_prefix` | Releases prefix for Harbor registry | No | `devops_sandbox_releases` |
| `is_release` | Whether this is a pre-release or release version | No | - |
| `dev_tools_version` | Dev tools version for devenv image | No | - |

## Secrets

| Secret | Description |
|--------|-------------|
| `GH_TOKEN` | GitHub token to clone utils-docker-platform repo |
| `REGISTRY_USERNAME` | Azure Container Registry username |
| `REGISTRY_PASSWORD` | Azure Container Registry password |
| `HARBOR_CLI_SECRET` | Harbor registry password |
| `HARBOR_USERNAME` | Harbor registry username |

## Image Naming Convention

Images are tagged in ACR as:
```
iqgeoproddev.azurecr.io/{engineering_prefix}/platform/platform-{image_type}:{build_id}_{arch}
```

Multi-arch manifests are created in Harbor as:
```
harbor.delivery.iqgeo.cloud/{engineering_prefix}/platform/platform-{image_type}:{version}
harbor.delivery.iqgeo.cloud/{releases_prefix}/platform/platform-{image_type}:{version}  (if is_release)
```

Where:
- `{image_type}` = base, build, appserver, tools, or devenv
- `{arch}` = arm-64 or amd-64
- `{version}` = Platform version (e.g., 7.4.0)

## Dependencies

This workflow uses:
- **[devops-engineering-ci-public-platform-build-push-action](https://github.com/IQGeo/devops-engineering-ci-public-platform-build-push-action)** - Builds and pushes individual architecture images
- **[devops-engineering-ci-public-multi-arch-action](https://github.com/IQGeo/devops-engineering-ci-public-multi-arch-action)** - Creates multi-arch manifests and pushes to registries
- **[utils-docker-platform](https://github.com/IQGeo/utils-docker-platform)** - Contains the dockerfiles (platform/7x/)

## Dockerfiles

All dockerfiles are located in the `utils-docker-platform` repository under `platform/7x/`:

- `dockerfile.base` - Base platform image
- `dockerfile.build` - Build tools image
- `dockerfile.appserver` - Application server image
- `dockerfile.tools` - Tools image
- `dockerfile.devenv` - Development environment image

## Build Process

Each architecture-specific build:
1. Checks out `utils-docker-platform` repository
2. Sets up Docker Buildx
3. Logs in to Azure Container Registry
4. Builds image with appropriate build arguments
5. Pushes to ACR with architecture-specific tag

Each multi-arch step:
1. Pulls both arm64 and amd64 images from ACR
2. Creates multi-arch manifest
3. Pushes manifest to ACR
4. Pushes manifest to Harbor (engineering and optionally releases prefix)
5. Tags with version and optional additional tags

## Example Output

A successful run produces images like:
```
# ACR (architecture-specific)
iqgeoproddev.azurecr.io/devops_sandbox_engineering/platform/platform-base:123_arm-64
iqgeoproddev.azurecr.io/devops_sandbox_engineering/platform/platform-base:123_amd-64

# Harbor (multi-arch manifests)
harbor.delivery.iqgeo.cloud/devops_sandbox_engineering/platform/platform-base:7.4.0
harbor.delivery.iqgeo.cloud/devops_sandbox_engineering/platform/platform-base:pre-release
harbor.delivery.iqgeo.cloud/devops_sandbox_engineering/platform/platform-base:latest
```

## Troubleshooting

### Build Failures

- **Check dependency order**: Ensure previous images completed successfully
- **Verify build args**: Each image type requires specific build arguments
- **Check registry credentials**: Ensure secrets are properly configured

### Missing Images

- **Check ACR**: Architecture-specific images should be in ACR first
- **Check Harbor**: Multi-arch manifests are created after both architectures complete
- **Verify prefixes**: Engineering vs releases prefix depends on `is_release` input

## Related Workflows

- **[devops-engineering-ci-public-build-platform-workflow](https://github.com/IQGeo/devops-engineering-ci-public-build-platform-workflow)** - Main platform build workflow that calls this workflow
