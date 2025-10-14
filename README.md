# docker-ceph-dovecot-build

Docker build contexts for developing, building, and testing Dovecot with Ceph (librados). This repo provides:

- A CI-friendly builder image for compiling Dovecot from source (used by Travis)
- A base build image consumed by Ceph builds
- A Ceph-only build image
- A “combined” development container bundling a Ceph demo cluster and Dovecot for plugin development

See also: `.travis.yml` and `.gitlab-ci.yml` for CI usage patterns.

## Repository layout

- `base/`
	- `Dockerfile`: Ubuntu 16.04 base with build toolchain; clones Dovecot sources. Intended to be tagged as `cephdovecot/travis-base` for downstream builds.
- `build/`
	- `Dockerfile`: Ubuntu 18.04 builder. Builds Dovecot from a branch/tag passed via `--build-arg DOVECOT`. Adds configs and test tools (imaptest, postfix).
	- `dovecot_config/`: Service units, example configs and test fixtures used in the image.
- `ceph/`
	- `Dockerfile`: Builds Ceph from source (ARG `CEPH`, default `v12.2.0`) on top of the base image.
- `combined/`
	- `Dockerfile`: CentOS-based image derived from `ceph/daemon` that compiles Dovecot and includes a Ceph demo entrypoint. Args: `DOVECOT` (default `2.3.15`), `CEPH` (default `v15.2.16`).
	- `entrypoint.sh`: Wraps Ceph container entrypoints; default daemon is `demo`. Useful for local development of the ceph-dovecot plugin.
	- `README.md`: Additional usage focused on plugin development.

## Features

- Build Dovecot from specific branches/tags (e.g., `release-2.2.21`, `master-2.3`, `master-2.2`)
- Ceph demo stack for rapid local testing (combined image)
- Includes imaptest, postfix, and sample configs for functional checks
- Non-root service users and pre-created directories for Dovecot runtime

## Build instructions

### CI builder image (from `.travis.yml`)

The CI job builds the image in `build/` with a Dovecot ref and publishes per-ref tags:

```
docker build --build-arg DOVECOT=$DOVECOT -t travis-build build
docker tag travis-build cephdovecot/travis-build-$DOVECOT:latest
docker push cephdovecot/travis-build-$DOVECOT:latest
```

Environment matrix used in CI:

- `DOVECOT=release-2.2.21`
- `DOVECOT=master-2.3`
- `DOVECOT=master-2.2`

Locally, supply your desired Dovecot ref:

```
docker build --build-arg DOVECOT=master-2.3 -t travis-build build
```

### Base image

Build the base image once if you need to consume it locally (some downstream images assume the tag name below):

```
docker build -t cephdovecot/travis-base:latest base
```

### Ceph build image

Build Ceph sources on top of the base image. You can override the Ceph version with `--build-arg CEPH=vX.Y.Z`.

```
docker build -t travis-ceph ceph
```

### Combined development image

This image compiles Dovecot on top of a `ceph/daemon` base and wires a Ceph demo. You can override `DOVECOT`/`CEPH` via build-args.

```
docker build -t ceph-dovecot-combined:dev combined
```

Run a local Ceph demo + Dovecot environment mounting your plugin sources:

```
docker run -d \
	--name ceph_dovecot_combined \
	--mount type=tmpfs,destination=/etc/ceph \
	-v "$(pwd)":/repo \
	-p 10143:10143 \
	-e MON_IP=127.0.0.1 \
	-e CEPH_PUBLIC_NETWORK=127.0.0.0/24 \
	-e CEPH_DEMO_UID=test_uuid \
	ceph-dovecot-combined:dev demo
```

Inside the container, follow the plugin build steps (see `combined/README.md`):

```
cd /repo
./autogen.sh
./configure --with-dovecot=/usr/local/lib/dovecot --enable-maintainer-mode --enable-debug --with-integration-tests --enable-valgrind --enable-debug
make clean install
chmod 777 /etc/ceph/*
chmod -R 777 /usr/local/var/
ldconfig
dovecot
```

You can run imaptest and connect to localhost using the mapped ports as needed.

## CI pipelines

- Travis (`.travis.yml`): builds `build/` for each `DOVECOT` ref and pushes to `cephdovecot/travis-build-$DOVECOT:latest`.
- GitLab CI (`.gitlab-ci.yml`): builds `combined/` and pushes to a configured registry using `${CI_COMMIT_SHA}` as the tag.

## Notes and caveats

- The images pin older base distributions (Ubuntu 16.04/18.04, CentOS in combined). Update only when requested, and adjust build dependencies accordingly.
- The combined image depends on `ceph/daemon` scripts and their environment variables; defaults are set in `combined/entrypoint.sh`.
- For production-like deployments, prefer official Ceph/Dovecot images; this repo targets development and CI.

