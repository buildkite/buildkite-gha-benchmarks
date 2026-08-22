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
[`workloads`](workloads). Kafka is the first executable provider lane and uses
[`scripts/run-kafka`](scripts/run-kafka).

| Target | Command |
| --- | --- |
| Kafka | `./gradlew --build-cache build -x test` |
| gRPC | `bazel build :grpc` |
| Mastodon | `docker buildx build --platform linux/amd64,linux/arm64` |
| PostHog | Mastodon command plus `BUILDKIT_CONTEXT_KEEP_GIT_DIR=1` |

## Run the canary

Create a Buildkite build for the same repository commit with
`BENCHMARK=kafka` to run the Buildkite lane. The pipeline imports
`.github/workflows/kafka.yml` using the pinned `github-actions`
Buildkite plugin and `buildkite-gha` release. It maps
`ubuntu-24.04` to the `hosted-m` queue because Kafka's upstream Gradle settings
request a 4 GiB daemon heap. The lane enables Gradle's local build cache and
transports Gradle User Home through the setup action. Builds without
`BENCHMARK=kafka` run only the static harness checks, so pushes and pull
requests cannot accidentally start the paid Kafka workload.

Each successful workload writes `benchmark-result.json`. Failed Gradle builds
also write a result and return the original failure status. A failure before
the benchmark script starts, such as Java setup or job provisioning failure,
is represented by the provider's job result instead.

## Cache modes

The Kafka lane currently measures cached/default operation. It enables the
local Gradle build cache and allows the setup action to restore and save Gradle
User Home. A cacheless mode must use a separate cache namespace and result
series; cached and cacheless results must not be combined.

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
