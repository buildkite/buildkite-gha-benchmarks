# buildkite-gha benchmarks

This repository runs pinned open-source workloads on Buildkite Hosted through
[`buildkite-gha`](https://github.com/buildkite/buildkite-gha). It provides a
stable performance canary and concrete workflows for hill-climbing GitHub
Actions compatibility on Buildkite.

The first workload is an engineering canary, not a published performance
claim: Apache Kafka at one exact commit, built with Java 21 and
`./gradlew --no-build-cache build -x test`. It establishes the source,
execution, timing, and result contracts before repetition, cache, cost, and
reporting automation are added.

The first workload uses [`scripts/fetch-source`](scripts/fetch-source) and
[`scripts/run-kafka`](scripts/run-kafka). The upstream repository and commit
are locked in [`benchmarks/lock.json`](benchmarks/lock.json), and the measured
command lives in [`workloads/kafka`](workloads/kafka).

## Run the canary

Create a Buildkite build for the same repository commit with
`BENCHMARK=kafka` to run the Buildkite lane. The pipeline imports
`.github/workflows/kafka.yml` using the pinned `github-actions`
Buildkite plugin and exact `buildkite-gha` source commit. Builds without that
environment value run only the static harness checks, so pushes and pull
requests cannot accidentally start the paid Kafka workload. The source pin can
return to a released runtime after the merged compatibility implementation is
released.

Each successful workload writes `benchmark-result.json`. Failed Gradle builds
also write a result and return the original failure status. A failure before
the benchmark script starts, such as Java setup or job provisioning failure,
is represented by the provider's job result instead.

## What “cacheless” means

The canary gives Gradle a fresh job-private `GRADLE_USER_HOME` and explicitly
disables Gradle's build cache. Dependency and Gradle distribution downloads are
therefore part of the measured command. It does not claim that provider-level
network, operating-system, or transparent storage caches are absent.

This mode is intentionally distinct from a future provider-recommended mode,
where each provider's normal remote caches and accelerators will be enabled.
The two modes must not be combined into one speedup figure.

## Validate locally

The lightweight checks do not clone or build Kafka:

```sh
scripts/check
```

The result contains bounded machine facts and timings, but no environment dump,
hostname, token, or secret. The scripts never execute an unpinned upstream
revision.
