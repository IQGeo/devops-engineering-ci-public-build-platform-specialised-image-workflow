# Who Calls What

## Scope

This document shows the inbound and outbound workflow dependencies for `devops-engineering-ci-public-build-platform-specialised-image-workflow`.

## Matrix

| Direction | Component | Relationship | Notes |
| --- | --- | --- | --- |
| Called by | `devops-engineering-ci-public-build-platform-workflow` | Inbound | The main platform workflow calls both specialised-image workflows |
| Calls | `devops-engineering-ci-public-platform-build-push-action/action.yml` | Direct | Builds one architecture-specific specialised platform image |
| Calls | `devops-engineering-ci-public-multi-arch-action/action.yml` | Direct | Creates and retags the manifest after architecture-specific builds complete |
| Contains | `.github/workflows/build-specialised-images.yml` | Internal | Orchestrates base, build, appserver, tools, and devenv image families |
| Contains | `.github/workflows/build-devdb-qa-images.yml` | Internal | Orchestrates the DevDB QA build chain |

## Notes

- This repo owns specialised platform-image orchestration rather than top-level platform release orchestration.
- Its workflows are not general-purpose callers; they sit under the main platform workflow.