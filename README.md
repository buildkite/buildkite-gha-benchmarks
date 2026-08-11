# buildkite-gha benchmarks

This repository measures the same pinned open-source workloads on GitHub
Actions, Depot runners, and Buildkite Hosted through
[`buildkite-gha`](https://github.com/buildkite/buildkite-gha).

The first workload is an engineering canary, not a published performance
claim: Apache Kafka at one exact commit, built with Java 21 and
`./gradlew build -x test`. It establishes the source, execution, timing, and
result contracts before repetition, cache, cost, and reporting automation are
added.

## Current lanes

| Lane | Runner | Workflow |
| --- | --- | --- |
| GitHub Actions | Public `ubuntu-24.04` (4 vCPU, 16 GB) | `.github/workflows/kafka-github-depot.yml` |
| Depot | `depot-ubuntu-24.04-4` (4 vCPU, 16 GB) | `.github/workflows/kafka-github-depot.yml` |
| Buildkite | Hosted queue via `buildkite-gha` | `.github/workflows/kafka-buildkite.yml` |

Every lane uses [`scripts/fetch-source`](scripts/fetch-source) and
[`scripts/run-kafka`](scripts/run-kafka). The upstream repository and commit
are locked in [`benchmarks/lock.json`](benchmarks/lock.json). Provider setup is
allowed to differ, but source selection and the measured command are shared.

## Run the canary

Prerequisites:

- the Depot GitHub App is installed for this public repository and its runner
  group permits public repositories;
- the `buildkite-gha-benchmarks` Buildkite pipeline is connected to this
  repository and targets the Hosted queue; and
- the selected commit is pushed before either provider is dispatched.

Run **Kafka / GitHub and Depot** manually from the repository's Actions page.
The one dispatch starts both jobs together.

Create a Buildkite build for the same repository commit with
`BENCHMARK=kafka` to run the Buildkite lane. The pipeline imports
`.github/workflows/kafka-buildkite.yml` using the pinned `github-actions`
Buildkite plugin and exact `buildkite-gha` source commit. Builds without that
environment value run only the static harness checks, so pushes and pull
requests cannot accidentally start the paid Kafka workload. The source pin can
return to a released runtime after its `actions/setup-java` compatibility fix
ships.

Each successful workload writes `benchmark-result.json`. Failed Gradle builds
also write a result and return the original failure status. A failure before
the benchmark script starts, such as Java setup or job provisioning failure,
is represented by the provider's job result instead.

## What “cacheless” means

The canary gives Gradle a fresh job-private `GRADLE_USER_HOME` and does not
enable Gradle's build cache. Dependency and Gradle distribution downloads are
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
