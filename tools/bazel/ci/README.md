# Shared Bazel CI helpers

## Agent AppArmor profile selection

The slave defaults to `SONIC_SLAVE_APPARMOR_PROFILE=unconfined`, preserving the
existing native-Docker and privileged/DinD behavior for Make and Bazel builds.
On an agent provisioned with the named `sonic-bazel-builder` profile, explicitly
select it with `SONIC_CONFIG_USE_NATIVE_DOCKERD_FOR_BUILD=y` and
`SONIC_SLAVE_APPARMOR_PROFILE=sonic-bazel-builder`. Pass these options consistently
to configure and subsequent builds. This branch's VS CI and main official-build
entrypoints explicitly set these options in their shared build options, so
configure and subsequent builds select the same profile. Their agents must be
provisioned with the profile before running these pipelines. Other pipeline
entrypoints and the shared configuration default remain unchanged.

The selector replaces only the AppArmor option in native-Docker mode; existing
capabilities and seccomp settings are unchanged. Custom profile selection with
privileged/DinD mode is rejected rather than ignored. An empty profile is also
rejected. Docker rejects a missing named profile at container startup; there is
no automatic fallback to `unconfined`.

Native-Docker slaves start with soft and hard open-file limits of 1048576,
matching `files/docker/docker` during image assembly. This lets the chroot Docker
startup set its required limit without granting `CAP_SYS_RESOURCE`. The agent's
`sysctl fs.nr_open` must be at least 1048576. Privileged/DinD slaves retain their
existing 524288 limits. Recreate the slave container after changing these limits.

The pipelines also set `SONIC_SLAVE_UNCONFINED_SYSTEM_PATHS=y`, which adds
`--security-opt systempaths=unconfined`. Docker's default protected `/proc`
submounts can prevent Bazel from mounting a new proc filesystem in its user/PID
namespace even when namespace creation itself succeeds. This option removes
Docker's masked and read-only system-path protections, including sensitive paths
under `/proc` and `/sys`; the named AppArmor profile does not replace them.
It is a security relaxation for approved build agents, not an isolation-equivalent
replacement. The shared default is `n`; unrelated pipelines/builds keep their
existing settings. Only `y` or `n` is accepted, and `y` requires native-Docker mode.
No extra agent sysctl or daemon change is needed for this option.

The named profile permits user namespaces using AppArmor's `default_allow` mode;
it does not add restrictive filesystem confinement. Installing it alone does not
change existing containers or the host-wide user-namespace restriction. First
verify that the agent keeps `kernel.apparmor_restrict_unprivileged_userns=1` and
has loaded the profile, then test inside the actual Trixie slave:

```sh
SONIC_RUN_CMDS='id; cat /proc/self/attr/current; unshare --user --map-current-user --mount --pid --fork true' \
  make -f Makefile.work BLDENV=trixie \
    SONIC_CONFIG_USE_NATIVE_DOCKERD_FOR_BUILD=y \
    SONIC_SLAVE_APPARMOR_PROFILE=sonic-bazel-builder \
    SONIC_SLAVE_UNCONFINED_SYSTEM_PATHS=y sonic-slave-run
```

The user must be non-root, its profile must name `sonic-bazel-builder`, and the
namespace probe must succeed. Then validate an actual uncached Bazel kernel build
with `linux-sandbox`. Normal Make builds remain supported, but named-profile use
still depends on the provisioned agent and successful native-Docker validation.
Also repeat the namespace probe with `--mount-proc` and execute Bazel's actual
`linux-sandbox` helper. In the host's `docker inspect` output, `MaskedPaths` and
`ReadonlyPaths` should be empty. A successful helper probe does not replace
validation of the full kernel build.

## Remote cache uploads

The Bazel OCI workflow enables remote action-cache uploads for the AMD64 and
ARM64 archive builds and the VS build only on pushes to `master`. Pull requests and manual runs remain
read-only. The archive job overrides the default in its ignored `.bazelrc.user`,
which the shared Bazel configuration imports without modifying tracked files;
the VS job forwards the same policy through `BAZEL_CONTAINER_ARGS` to the Make
builder and source validation commands.
The agents and builder containers must be able to reach the remote cache, and
the server must authorize writes. Enforce writer permissions on the server:
the workflow flag is not an authorization boundary for pull-request code.
Enabling uploads does not guarantee backfilling results already present in
local caches.

The fresh-checkout disk-cache verifier explicitly disables the remote cache,
remote executor, and remote downloader for its isolated builds. This keeps remote
hits from masking disk-cache misses and avoids enabling the remote downloader
without the gRPC cache it requires. Normal workflow builds retain their configured
remote services.

If an isolated build fails, the verifier prints its retained Bazel build log
and exit code to the CI console before raising the failure. The local
`agent-cache/<phase>/build.log` remains available on the runner; raw logs are
not added to public failure artifacts.

## Debian download mirrors

The root `.bazelrc` selects [`debian_mirrors.cfg`](../debian_mirrors.cfg) for
Bazel-managed dependency downloads, including image packages and published
toolchain inputs. The pinned upstream Debian snapshot `20260727T143429Z` maps
to the SONiC mirror snapshot `20260727T000818Z`; security snapshot
`20260726T121236Z` maps to `20260727T000824Z`. Original upstream URLs remain
fallback candidates. Existing image-lock mirror URLs are normalized to HTTPS,
and their security snapshot `20260726T000750Z` maps to `20260727T000824Z`.
Other snapshot timestamps retain their upstream location.

The policy changes download locations, not generated locks, package versions
or checksum verification. When advancing snapshot inputs, verify the matching
mirror content before updating the mappings. Commands that ignore rc files
must pass `--downloader_config` explicitly. The policy does not rewrite
`apt-get` or arbitrary `curl` commands, and does not provide connectivity to the
private remote cache/downloader. The mirror may redirect HTTPS requests to
HTTP, so HTTPS source URLs alone do not guarantee end-to-end TLS.

## Shared helpers

These helpers validate buildimage artifacts and preserve build evidence. The
shared container runner discovers the selected owner's Bazel CI targets;
package expectations remain in the owner declarations and tests.

[`.github/workflows/bazel-oci.yml`](../../../.github/workflows/bazel-oci.yml) is
the shared entrypoint for Bazel OCI container CI. It runs common AMD64/ARM64
archive checks, lightweight hosted regressions and the full VS integration
job. SWSS and Syncd source-package checks run after Make in that VS job, using
its checkout, Trixie builder, platform/feature flags and persistent Bazel action
cache. The private builder output bases remain separate; cached outputs can
be restored without recompiling the same source packages. Add checks for future
OCI containers in this workflow, keeping their target lists and package contracts with each container.
Keep checks that compile image inputs in the full build job so they can reuse
its action outputs.
The owning BUILD files sit beside the original container inputs:
[`dockers/docker-orchagent/BUILD.bazel`](../../../dockers/docker-orchagent/BUILD.bazel)
and [`platform/vs/docker-syncd-vs/BUILD.bazel`](../../../platform/vs/docker-syncd-vs/BUILD.bazel).
The shared runner accepts either package path; it does not require owners to
live under `dockers/`.
Manual runs default to `skip_vs=true`; same-repository PRs retain the full VS
job after hosted regressions and dependency resolution pass. Fork PRs and
manual `skip_vs=true` runs omit source-package/full-image validation; hosted
success alone does not establish those results.

The native VS job reuses `target/docker-orchagent.gz` produced as an installer
dependency. `build_vs_with_reused_orchagent.sh` captures that archive after the
successful VS build, requests the remaining artifacts, and verifies the retained
runtime archive against its Make metadata and the matching debug image. Missing,
changed or incompatible archives fail the job; the script does not fall back to
an unnoticed rebuild. Standalone Make archive targets keep their normal input
and builder-switch checks.

To measure the avoided request on the same revision, runner, configuration and
warmed caches, dispatch `Bazel OCI` with `skip_vs=false` and
`measure_orchagent_reuse=true`. This calibration deliberately executes the old
request once and requires the runtime archive to remain identical. The local
`artifacts/vs/orchagent-reuse-timing.json` reports its measured wall time, the new
validation cost and their difference. This measures the removed request, not a
paired full-job speedup. Normal PR runs omit the request. Existing source,
installed-image, debug-symbol and firmware boot checks remain required.

- `container.py` runs an owner's `:ci_tests` test suite and builds its
  `:ci_artifacts` filegroup. Tests and exported files are declared in the
  container's `BUILD.bazel`; no Python CI configuration or callback is loaded.
  For example, for separate cold runs after preparing the matching Make
  manifests (use separate clean workspaces for these examples):

  ```sh
  python3 -B tools/bazel/ci/container.py \
    --container //dockers/docker-orchagent --source-file rules/docker-orchagent.mk --bazel bazel --artifacts artifacts/swss
  python3 -B tools/bazel/ci/container.py \
    --container //platform/vs/docker-syncd-vs --source-file platform/vs/docker-syncd-vs.mk \
    --bazel-arg=--config=syncd-dash --bazel bazel --artifacts artifacts/syncd-vs/bazel
  ```

  Start from native AMD64 Trixie without `MODULE.bazel.lock` and use an empty
  artifact directory. To validate after Make, pass `--reuse-build`: it requires
  the existing nonempty, ignored and untracked generated lock, records its
  initial hash, and retains the tested lock/graph afterward. It never deletes
  the lock, clears outputs or disables package/source checks. Use Make's exact
  feature flags (currently `--//dockers/docker-orchagent:source_config_engine=true`
  for FIPS SWSS, `--config=syncd-dash` for Syncd), both native platform flags,
  and the same cache/download options. In this mode, the Git cleanliness check
  covers the declared Bazel inputs and their containing submodules, allowing
  unrelated legacy Make submodules to create their normal build patches.
  Changes to validated inputs or their submodule commits still fail.
  In reuse mode the artifact group is built before any tests, using Bazel's
  execution log to record cache hits and executed actions by target and mnemonic.
  The raw log is private and deleted after extracting those counts; no command,
  environment or file-path fields are published. This distinguishes reuse of
  Make's production outputs from later test-only compilation. Cache misses are
  reported for inspection rather than hidden or treated as a test failure.
  Reuse Make's generated manifests rather
  than rerendering them with different defaults. The runner expands nested test
  suites, audits their actual targets and the artifact group before execution, runs uncached tests, collects
  configured logs/XML and package-check summaries, and exports the group's files
  under their declared basenames. Duplicate output filenames fail; an empty
  artifact group is valid for contract-only CI. Dependency resolution and source
  hashes are retained with success/failure receipts. `--bazel-arg=...` appends an
  explicit build option override. `--source-file PATH` adds a workspace-relative
  preparation input to the source evidence; the workflow supplies the same
  owning Make include that prepares the manifests.

  SWSS's actual source-layer validation is a Bazel test with declared archives,
  owner install files and binutils tools. Syncd's suite validates its existing
  Make/source package adapter. Both suites use the same manifest-test program
  and the shared OCI package/image regression tests. Adding a test or artifact
  requires changing the owning BUILD declaration, with no second list in the
  runner or public artifact test-count checks.
- `artifact_validation.py` streams tar contents to record ownership, modes, file
  hashes and little-endian ELF64 metadata, and checks matching split debug symbols.
  SWSS and syncd share its ELF inspection and debug-link checks; each consumer
  supplies its package inventory and platform policy. SWSS's inventory and
  Protobuf expectations are declared in its BUILD-generated `:package_policy`
  and interpreted by `tools/bazel/oci/package_contract.py`.
  `debug_archives` combines a runtime tar and its symbols tar, rejects payload
  collisions and foreign debug paths, checks the caller's ELF architecture,
  safely extracts the pair and verifies matching build IDs, DWARF and debug links.
- `build.py` builds a caller's filename-to-label mapping and collects one nonempty
  archive per label, with atomic publication, command logs, sizes and hashes.
  `collect_artifacts` exports all files from a declared CI filegroup. It checks
  every output before publication and records each file's size, hash and Bazel
  output path. Owner package checks run as Bazel tests with declared source inputs.
  A package test records every validated archive's basename and SHA256 in
  `validation.artifact_sha256`. The runner and public upload collector require
  the exported names and bytes to match. The source-layer test and artifact filegroup use identical build options
  so their inputs and exported bytes match.
- `command_log.py` captures command output and timings while keeping diagnostic
  messages separate from queried artifact paths. Callers supply the working
  directory and evidence directory. The shared runner sends raw action-graph
  stdout to a temporary private file, retains only stderr in the log and removes
  the raw graph after its execution audit, including on failure.
- `public_artifacts.py` prepares explicit public upload lists for the archive,
  SWSS source, VS and Syncd contract evidence. The three source/image artifact
  selections now run in the VS job; the successful VS summary also requires
  matching successful SWSS and Syncd summaries. Container jobs compare the expanded
  suite identities with successful build-event results instead of maintaining
  a second fixed test count. Successful jobs retain their expected
  build outputs and validated dependency JSON, plus summaries containing only
  declared status, count, revision and hash fields. Failed jobs may retain only a
  partial summary. The helper excludes raw command, Bazel event/execution and
  test logs/XML from uploads; these records remain in the job workspace.
  It rejects missing successful outputs, symlinked inputs, nonempty recorded
  environments, diagnostic command fields, credential-bearing URLs, common token
  formats and known host build or
  cache paths. The owning build step remains responsible for validating binary
  payloads. Update this helper's declared paths, test counts and receipt fields
  with the owning workflow when its outputs change. The helper does not run a
  build, and its direct tests use synthetic records only.
- `python_packages.py` exports the config-engine and py-common wheels and retains
  source-suite logs/XML, collected-test inventories and the installed-wheel
  receipt. It requires matching wheel hashes, native architecture and Python
  version, and fails when any required evidence is missing.
- `resolution.py` retains the generated module lock and resolved graph. Its
  explicit allowlist for known unused-extension failures is tied to the selected
  dependency versions and rejects incomplete graphs or unexpected failures.
- `verify_agent_cache.py` builds the shared OCI archive fixture across fresh
  checkouts and output bases, verifies disk-cache hits, then changes an input and
  requires a rebuild. It shares cache settings through `tools/bazel/build_helpers.py`.

Run the helper tests from the repository root:

```sh
python3 -B -m unittest discover -s tools/bazel/tests -p '*_test.py'
```

The ELF tests use `cc`, `readelf` and `objcopy`. The cache integration check uses
the repository's selected Bazel version and platform; its artifact directory
must be new:

```sh
python3 -B tools/bazel/ci/verify_agent_cache.py \
  --bazel bazel --artifacts artifacts/agent-cache
```

The workspace rc uses the [maintained registry selection](../README.md#registry-selection).
Resolution receipts retain the actual registry URLs from the generated lock
instead of inferring them from the default `.bazelrc`.

## Public build evidence

Container CI uses `public_artifacts.py` from the public upload work in
[Image12](https://github.com/securely1g/sonic-buildimage/pull/12). It uploads
explicit build outputs, checked dependency records, and summaries of build and
test results. Raw Bazel events, command lines, logs, and receipt diagnostics stay
out of public artifacts. A failed job publishes only a validated partial summary.
The selector fails closed for unsafe dependency data or missing successful outputs.
