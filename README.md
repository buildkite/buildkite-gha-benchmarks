# buildkite-gha benchmarks

This repository runs pinned open-source workloads on Buildkite Hosted through
[`buildkite-gha`](https://github.com/buildkite/buildkite-gha). It provides a
stable performance canary and concrete workflows for hill-climbing GitHub
Actions compatibility on Buildkite.

The repository pins four engineering targets: Apache Kafka, gRPC, Mastodon,
and PostHog. The target commands build Kafka with Gradle, gRPC with Bazel, and
multi-platform Mastodon and PostHog container images. These are workload
contracts, not published performance claims.

Upstream repositories and commits are locked in
[`benchmarks/lock.json`](benchmarks/lock.json). Exact measured commands live in
[`workloads`](workloads). Purpose-built workflows under
`.github/workflows` preserve each target's toolchain, platforms, and cache
behavior.

| Target | Command |
| --- | --- |
| Kafka | `./gradlew --build-cache build -x test` |
| gRPC | `bazel build :grpc` |
| Mastodon | `docker buildx build --platform linux/amd64,linux/arm64` |
| PostHog | Mastodon command plus `BUILDKIT_CONTEXT_KEEP_GIT_DIR=1` |

## Run the canary

Create a Buildkite build for the same repository commit with `BENCHMARK` set to
`kafka`, `grpc`, `mastodon`, or `posthog`. The selected lane imports its matching
workflow using the pinned `github-actions` Buildkite plugin and
`buildkite-gha` release. All four workflows map `ubuntu-24.04` to the
`hosted-m` queue. Builds without a recognized `BENCHMARK` value run only the
static harness checks, so pushes and pull requests cannot accidentally start a
paid workload.

The Kafka and gRPC scripts write `benchmark-result.json` and return the original
build status. Container-build timing comes from the executable job because the
Docker build action owns that operation. A failure before a benchmark starts,
such as tool setup or job provisioning failure, is represented by the provider's
job result.

## Cache modes

Each lane measures cached/default operation. Kafka transports Gradle User Home,
gRPC enables Bazelisk, disk, and repository caches, and the container workflows
use BuildKit's GitHub Actions cache backend with maximal export. A cacheless
mode must use a separate cache namespace and result series; cached and cacheless
results must not be combined.

Provider-level network, operating-system, and transparent storage caches may
still affect either mode.

## Validate locally

The lightweight checks do not clone or build upstream projects:

```sh
scripts/check
```

The result contains bounded machine facts and timings, but no environment dump,
hostname, token, or secret. The scripts never execute an unpinned upstream
revision.
