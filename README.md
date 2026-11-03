# buildkite-gha benchmarks

This repository runs pinned open-source workloads on Buildkite Hosted through
[`buildkite-gha`](https://github.com/buildkite/buildkite-gha). It provides a
stable performance canary and concrete workflows for hill-climbing GitHub
Actions compatibility on Buildkite.

The repository pins four engineering targets: Apache Kafka, gRPC, Mastodon,
and PostHog. The target commands build Kafka with Gradle, gRPC with Bazel, and
multi-platform Mastodon and PostHog container images.

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

Run any workflow manually from the GitHub Actions UI. To run the same workflow
on Buildkite, create a build for the same repository commit with `BENCHMARK` set
to `kafka`, `grpc`, `mastodon`, or `posthog` and
`BUILDKITE_GITHUB_EVENT=workflow_dispatch`. Without the explicit event, manual
API builds use `push` context and these workflows are skipped. For example:

```sh
bk build create -p buildkite/buildkite-gha-benchmarks \
  -b "$BRANCH" -c "$SHA" \
  -e BENCHMARK=kafka -e BUILDKITE_GITHUB_EVENT=workflow_dispatch \
  --yes --no-input
```

The selected lane imports the matching workflow using the pinned
`github-actions` Buildkite plugin and
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

For a fresh comparison, change Kafka's `GRADLE_BUILD_ACTION_CACHE_KEY_PREFIX`
and both gRPC's `cache-version` and `disk-cache` to unused generations. Keep
Kafka's strict cache matching and gRPC's absolute source path and compiler paths.
Run the same prime harness commit on both providers, verify initial misses and
completed cache saves, then change only the source locks for the target commit.
Verify task/action hits, not only archive restoration. Never reuse a generation
for an independent repetition.

## November 3, 2026 results

These runs use `buildkite-gha` v0.119.0 and plugin v0.14.0. Both providers ran
the same [prime harness commit](https://github.com/buildkite/buildkite-gha-benchmarks/commit/4d5d37c87f1a8eb1676056a6f28c2ad1e7238aa5),
completed cache saves, then ran the same [source-lock-only target commit](https://github.com/buildkite/buildkite-gha-benchmarks/commit/989065716fa431fa235e9861a13ac1b068cf8c58).
Times show workload seconds followed by complete executable-job seconds in
parentheses. Queue time and the separate Buildkite importer are excluded.

### Kafka result

The source transition remains `f311e621` → `e28414ee`, matching the
[August baseline](https://github.com/buildkite/buildkite-gha-benchmarks/blob/214ec90e435dd23dfa9b0397b6bfe9ce070de7f4/README.md#kafka-result).
Both providers reported identical task outcomes.

| Phase | GitHub Actions | Buildkite | Executed | Cached | Up-to-date |
| --- | ---: | ---: | ---: | ---: | ---: |
| Prime | [624.188 (667)](https://github.com/buildkite/buildkite-gha-benchmarks/actions/runs/41576693275) | [309.863 (367.689)](https://buildkite.com/buildkite/buildkite-gha-benchmarks/builds/81) | 755 | 2 | 2 |
| Target | [161.134 (203)](https://github.com/buildkite/buildkite-gha-benchmarks/actions/runs/41578808733) | [111.656 (242.309)](https://buildkite.com/buildkite/buildkite-gha-benchmarks/builds/84) | 388 | 305 | 66 |

Buildkite's target workload was 1.44× faster, but its complete job took 19.4%
longer. Gradle setup/cache restore took 75.378s versus about 9s on GitHub;
the 928 MB dependency archive alone took 62.511s to restore. Gradle post-action
work took 41.154s versus about 17s. The separate importer took 8.567s.

### gRPC result

The source transition remains `80eb9754` → `c9bdb307`, matching the
[August baseline](https://github.com/buildkite/buildkite-gha-benchmarks/blob/214ec90e435dd23dfa9b0397b6bfe9ce070de7f4/README.md#grpc-result).
Both providers reported identical Bazel process outcomes.

| Phase | GitHub Actions | Buildkite | Disk hits | Local | Internal |
| --- | ---: | ---: | ---: | ---: | ---: |
| Prime | [1100.963 (1174)](https://github.com/buildkite/buildkite-gha-benchmarks/actions/runs/41576697072) | [391.556 (462.096)](https://buildkite.com/buildkite/buildkite-gha-benchmarks/builds/82) | 0 | 2,013 | 283 |
| Target | [49.067 (132)](https://github.com/buildkite/buildkite-gha-benchmarks/actions/runs/41578813431) | [15.681 (89.998)](https://buildkite.com/buildkite/buildkite-gha-benchmarks/builds/85) | 2,012 | 1 | 283 |

Buildkite's target workload was 3.13× faster and its complete job was 1.47×
faster. Source fetch took 37.060s versus about 67s on GitHub; Bazel setup/cache
restore took 32.763s versus about 10s. The separate importer took 8.100s.

### Scope and measurement limits

- Each workload has one independent prime→target transition, not an aggregate.
- Both providers reported four CPUs, approximately 16 GiB RAM and Ubuntu 24.04.
  Buildkite reported `AMD EPYC`; GitHub targets reported `AMD EPYC 9V45`.
  CPU model and kernel parity were not established.
- Java 21.0.13, Gradle 9.7.1, Bazel 9.3.1 and GCC/G++ 13.3.0 matched across
  providers. Java and Bazel changed since August, so this is not a runtime-only
  before/after comparison.
- GitHub reported forcing Node 20 actions onto Node 24. Kafka cache commands used
  zstd on GitHub and gzip on Buildkite. Neither difference proves a speedup
  opportunity without a separate controlled experiment.
- The scripts record CPU model and compiler versions in `benchmark-result.json`.
  Use the external run identity for the provider: Buildkite hides ambient
  `BUILDKITE_*` variables, so the result's `provider` field can say `github-actions`.
  gRPC fetches source before the timed script; its JSON fetch duration only
  measures source validation. Use job-step timestamps for the actual fetch.

The next performance experiments should target cache restore/transport first,
then restore-only Kafka targets. No optimization was tested in this series.

## Validate locally

The lightweight checks do not clone or build upstream projects:

```sh
scripts/check
```

The result contains bounded machine facts and timings, but no environment dump,
hostname, token, or secret. The scripts never execute an unpinned upstream
revision.
