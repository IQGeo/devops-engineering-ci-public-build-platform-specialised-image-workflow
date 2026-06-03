# devops-engineering-ci-public-build-platform-specialised-image-workflow

Reusable GitHub Actions workflows for building the specialised IQGeo Platform image families.

## Overview

This repository contains the reusable workflows that build the main specialised platform images and the DevDB QA image chain. It sits between the top-level platform workflow and the low-level actions that perform architecture-specific builds and multi-arch manifest publication.

For inbound and outbound dependency relationships, see [docs/WHO-CALLS-WHAT.md](docs/WHO-CALLS-WHAT.md).

## Workflows

### `.github/workflows/build-specialised-images.yml`

Builds a caller-supplied set of specialised platform image types.

Key responsibilities:

- Accepts a JSON array of `image_types`.
- Builds amd64 and arm64 variants in parallel for each requested image type.
- Uses `devops-engineering-ci-public-platform-build-push-action` for the architecture-specific image builds.
- Uses `devops-engineering-ci-public-multi-arch-action` to publish and retag the manifest.
- Filters `-clear` image variants out of release retagging.

Current platform callers use it for these image groups:

- `base` and `base-clear`
- `build` and `build-clear`
- `appserver` and `tools`
- `devenv-2` and `devenv-2-clear`

### `.github/workflows/build-devdb-qa-images.yml`

Builds the DevDB QA image chain used for internal platform testing.

Key responsibilities:

- Builds `platform-devdb-build` first.
- Builds `platform-devdb-appserver` and `platform-devdb-tools` after that.
- Builds `platform-devdb-qa-appserver` as the final wave.
- Creates a multi-arch manifest after each image wave.

## Important inputs

For `build-specialised-images.yml`, the main inputs are:

- `version`: platform version being built.
- `image_types`: JSON array of requested image types.
- `updated_tags`: extra tags to apply to the final manifest.
- `shortened_version`: version shorthand for language pack lookup.
- `build_id`: run-specific identifier for architecture-specific tags.
- `engineering_prefix` and `releases_prefix`: target registry prefixes.
- `is_release`: controls whether release repositories are retagged.
- `dev_tools_version`, `preqs_tag`, and `pip_flags`: build arguments used by some image types.

For `build-devdb-qa-images.yml`, the main inputs are:

- `version`
- `modules`
- `updated_tags`
- `build_id`
- `engineering_prefix`
- `releases_prefix`
- `is_release`

## Build flow

Typical specialised image flow:

```text
build-specialised-images.yml
  -> build-arch-images via platform-build-push-action
  -> create-multi-arch-manifests via multi-arch-action
```

Typical DevDB QA flow:

```text
build-devdb-qa-images.yml
  -> devdb-build
  -> devdb-appserver and devdb-tools
  -> devdb-qa-appserver
  -> manifest publication after each wave
```

## How this repo fits the wider build stack

- Upstream: `devops-engineering-ci-public-build-platform-workflow` calls these workflows.
- Downstream: the workflows rely on `devops-engineering-ci-public-platform-build-push-action` and `devops-engineering-ci-public-multi-arch-action`.
- Scope: this repo owns specialised platform image orchestration, not top-level platform release logic.
