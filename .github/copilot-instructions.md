# Copilot instructions for docker-ceph-dovecot-build

This repository hosts Docker build contexts for developing, building, and testing Dovecot with Ceph (librados) across a few distinct images. Use these guidelines when assisting in this repo.

## Project intent
- Provide reproducible Docker images for CI and local development:
  - A Travis/GitHub CI-friendly builder that compiles Dovecot with librados
  - A base image used by Ceph builds
  - A Ceph-only build image
  - A “combined” development container that bundles a Ceph demo cluster and a Dovecot build for plugin work

## Key directories and roles
- base/Dockerfile: Ubuntu 16.04 base image (tagged externally as cephdovecot/travis-base) with build toolchain and Dovecot sources cloned.
- build/Dockerfile: Ubuntu 18.04 builder image used in CI to compile Dovecot, add configs and tools (imaptest, postfix). Build arg: DOVECOT.
- ceph/Dockerfile: Builds Ceph from sources (ARG CEPH) on top of cephdovecot/travis-base.
- combined/Dockerfile: CentOS-based image derived from ceph/daemon with Dovecot compiled in and a Ceph demo entrypoint; used for local dev of the ceph-dovecot plugin. Args: DOVECOT, CEPH.
- combined/entrypoint.sh: Entrypoint that wires up the Ceph demo scenarios and environment defaults.
- .travis.yml: Matrix builds of build/ image for several DOVECOT refs; logs in and pushes images.
- .gitlab-ci.yml: Simple pipeline that builds and pushes the combined image to a registry.

## Typical tasks Copilot should handle
- Update Docker build args (e.g., bump DOVECOT or CEPH) consistently across relevant Dockerfiles and docs.
- Adjust CI matrix in .travis.yml when adding/removing supported DOVECOT build refs.
- Improve Dockerfiles (layering, pinning, minor fixes) without breaking existing behavior; keep base OS/distros as-is unless requested.
- Update READMEs when images/features change; keep commands consistent with .travis.yml and combined/README.md.
- Keep shell scripts compatible with bash in this repo; avoid introducing zsh-only features in container scripts.

## Conventions and constraints
- Don’t remove or change entrypoint.sh behavior in combined/ without strong reason; many instructions assume the Ceph “demo” scenario.
- Maintain the .travis.yml flow for the build/ image:
  - docker build --build-arg DOVECOT=$DOVECOT -t travis-build build
  - docker tag travis-build cephdovecot/travis-build-$DOVECOT:latest
  - docker push cephdovecot/travis-build-$DOVECOT:latest
- base/Dockerfile uses a multi-stage alias (AS travis-base). When documenting or building locally, tag that result as cephdovecot/travis-base for use by ceph/.
- combined/ depends on ceph/daemon images; keep compatibility with the pinned base tag unless requested to upgrade.

## Style
- Dockerfiles: combine apt/yum operations to reduce layers; keep DEBIAN_FRONTEND=noninteractive where used; avoid leaking credentials.
- Shell: set -euo pipefail for new scripts (unless existing logic depends on exit codes); use bash.
- Docs: prefer concise, copy-pasteable commands with explanatory comments. Use fenced code blocks.

## Testing and validation
- For changes to Dockerfiles, demonstrate a local build command and note required build-args.
- For combined/, show a minimal ‘docker run … demo’ example and verify Dovecot binaries are present.
- If CI config changes, mirror .travis.yml/.gitlab-ci.yml patterns and update README accordingly.

## When in doubt
- Reflect current behavior in .travis.yml and combined/README.md.
- Avoid distro changes or major dependency upgrades without an explicit request.
- Keep edits minimal and scoped; update docs alongside any behavioral change.
