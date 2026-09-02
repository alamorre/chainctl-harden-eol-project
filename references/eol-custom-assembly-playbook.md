# EOL and pinned-project Custom Assembly playbook

Use this reference when implementing or reviewing a project. Adapt every package, artifact, user, entrypoint, and test to the pinned upstream revision; the Airflow example at the end is evidence, not a template.

## Outcome model

```text
verified upstream tag or commit + verified external inputs
                            |
                 inspect the pinned build
                            |
                  map native dependencies
                    /               \
          builder assembly       runtime assembly
          tools + headers        shared libraries
                    \               /
             digest-pinned multi-stage build
                            |
             per-architecture validation
                            |
        identity + integration + scan + evidence
```

The objective is a controlled rebuild of a fixed application revision, not an application upgrade. A modern OS layer reduces OS-package exposure; it does not confer upstream support or repair frozen application dependencies.

## 1. Intake and stopping decisions

Record these inputs before implementation:

| Input | Required decision or evidence |
|---|---|
| Project | Canonical GitHub repository URL and redistribution terms |
| Target | Full commit, signed release tag, or exact version |
| Reason | EOL requirement, compatibility constraint, rollback, or other need |
| Architectures | For example `linux/amd64` and `linux/arm64` |
| Deployment | Entrypoint, command, port, health check, user, writable paths |
| Runtime | Version supported by the pinned revision and available from Chainguard |
| Features | Extras, providers, plugins, database clients, optional modules |
| Services | Database, executor, queue, secrets backend, object store, Kubernetes |
| Chainguard source | Entitled image appropriate for extension |
| Organization | Directly confirmed by the user in the current session |
| Output names | Distinct builder and runtime custom repository names |
| YAML filenames | Both filenames chosen by the user before creation |
| Acceptance | Exact smoke and production-shaped tests |

Stop and ask for direction if:

- no immutable source identity can be established;
- a required download or submodule cannot be verified;
- redistribution rights or notice obligations are unclear;
- the required runtime is outside the revision's supported range;
- a required entitled package is missing or ABI-incompatible;
- success would require mixing unsupported package repositories;
- an application/dependency upgrade would violate the pin;
- required production behavior cannot be identified; or
- critical residual risk has no accepting owner.

## 2. Preserve and inventory the workspace

Run read-only inspection before changing the project:

```bash
git status --short
git rev-parse --show-toplevel
git remote -v
git log -1 --decorate --format=fuller
git rev-parse --is-shallow-repository
git submodule status --recursive
rg --files -g 'AGENTS.md' -g 'Dockerfile*' -g '*.lock' \
  -g '*requirements*' -g '*.yaml' -g '*.yml' -g 'LICENSE*' -g 'NOTICE*'
```

Record existing changes, detached or shallow state, release and dependency files, build workflows, image references, and architecture assumptions. Never discard unrelated changes. If a separate checkout is needed, use a dedicated directory or worktree and record its source.

Treat downloaded code as hostile until inspected. Look for install hooks, Make targets, CI scripts, Docker build steps, submodule URLs, generated downloads, credential access, network use, and privileged mounts. Execute builds in a disposable environment without ambient credentials or the host container socket.

## 3. Establish source and release-input identity

### Annotated release tag

```bash
git cat-file -t "${TARGET_TAG}"
git rev-parse "${TARGET_TAG}"
git rev-parse "${TARGET_TAG}^{}"
git verify-tag "${TARGET_TAG}"
```

Record both the tag object and peeled commit. A cryptographically valid signature only proves that the tag matches the available signing key; report trust warnings rather than claiming the signer is trusted without evidence.

### Untagged or unsigned revision

Record the full 40-character commit, nearest tag and distance, selection reason, and absence of signed-release evidence. Prefer canonical upstream Git data or checksummed upstream archives over third-party mirrors.

### External inputs

Inventory and pin constraints files, lockfiles, submodules, patches, generated sources, archives, downloaded binaries, and package-manager bootstraps. Verify downloads during the build:

```dockerfile
ARG INPUT_SHA256=<expected-sha256>
RUN curl -fsSL "<immutable-url>" -o /tmp/input \
    && echo "${INPUT_SHA256}  /tmp/input" | sha256sum -c -
```

Fail the build if the fetched commit, checksum, runtime, or built version differs from the recorded target. A lockfile with artifact hashes supports a stronger reproducibility claim than version-only constraints.

## 4. Reconstruct the pinned upstream build

Inspect the target revision's Dockerfile, entrypoint, CI/release workflows, runtime compatibility matrix, default extras/plugins, native packages, user/group behavior, paths, signals, ports, health checks, locale/timezone/CA assumptions, database initialization, and architecture branches.

Prefer documentation from the pinned release. Current `main` may describe dependencies and behavior that did not exist at the target commit.

Build a dependency map before writing assembly YAML:

| Application need | Builder | Runtime | Minimum validation |
|---|---|---|---|
| Native extension | compiler, linker, libc and language headers | matching shared libraries | build plus runtime import and `ldd` |
| PostgreSQL | client development headers | client/shared library | import and disposable connection |
| MySQL/MariaDB | connector development package | matching connector library | import and disposable connection |
| LDAP | LDAP/SASL/Kerberos headers | aligned LDAP/SASL/Kerberos libraries | import and dynamic-link check |
| ODBC | ODBC development package | ODBC runtime and required driver | import and driver load |
| Geospatial | geospatial headers | matching runtime libraries/data | import and small operation |
| TLS | crypto headers only when compiling | CA bundle and crypto libraries | controlled TLS request |

Use the organization's entitlements and package picker or current CLI output as authoritative. Package names and versions can change. Never add Wolfi, Alpine, Debian, or arbitrary third-party APK repositories to fill a Chainguard OS Custom Assembly gap.

## 5. Design builder and runtime assemblies

Use an entitled language/application image when it supports the pinned app and required native packages; use an entitled base when precise control is required.

Builder category:

- shell, source-control client, compiler/linker toolchain, and `pkg-config` support;
- language runtime and development headers;
- package-manager bootstrap tools pinned where possible; and
- justified native development packages.

Runtime category:

- language runtime or smaller runtime counterpart;
- CA certificates and a minimal init process when required;
- aligned shared libraries and data files;
- database clients and operational tools only when runtime behavior needs them; and
- no compiler, linker, development headers, source-control client, or package bootstrap tooling unless explicitly justified.

Custom Assembly is additive: source-image packages cannot be removed. A single assembly is acceptable only for an already self-contained artifact; record why it is sufficient.

After the user chooses the filenames, start with the smallest justified YAML:

```yaml
contents:
  packages:
    - ca-certificates
    - <language-runtime>
    - <only-justified-packages>
```

Custom Assembly can also configure environment, annotations, accounts, certificates, and runtime repositories. Keep all values non-sensitive, quote numeric environment values as strings, respect reserved prefixes, and verify the current schema with CLI help and official documentation.

Review before apply:

```bash
yamllint <builder-file.yaml> <runtime-file.yaml>
git diff --check -- <builder-file.yaml> <runtime-file.yaml>
```

Use a documented package-discovery loop:

1. Start only with packages justified by the pinned upstream build.
2. Apply the builder assembly and capture the complete Factory result.
3. Map each missing header, library, executable, or `pkg-config` module to an entitled package.
4. Add only that package and keep builder/runtime ABI families aligned.
5. Rebuild until the application builds and the runtime starts.

Do not change the pinned application or dependency graph to conceal an OS-package gap.

## 6. Create Custom Assembly repositories with `chainctl`

Before authenticated work:

```bash
command -v chainctl
chainctl update
chainctl auth login
chainctl auth status
chainctl images repos build apply --help
chainctl images repos build list --help
chainctl images repos build logs --help
```

The user must state the organization in this session before any organization-targeted command. Store it only in a task-specific variable such as:

```bash
CONFIRMED_ORG='<user-confirmed-organization>'
```

Create new builder and runtime repositories from the entitled base:

```bash
chainctl images repos build apply \
  --parent "${CONFIRMED_ORG}" \
  --repo '<entitled-source-repo>' \
  --file '<chosen-builder-file.yaml>' \
  --save-as '<new-builder-repo>' \
  --yes

chainctl images repos build apply \
  --parent "${CONFIRMED_ORG}" \
  --repo '<entitled-source-repo>' \
  --file '<chosen-runtime-file.yaml>' \
  --save-as '<new-runtime-repo>' \
  --yes
```

`--save-as` is single-repository only. Do not use it in batch mode. Do not update the entitled source image.

For an explicitly authorized update to an existing custom repository, omit `--save-as` and target that custom repo. Preview drift first:

```bash
chainctl images repos build apply \
  --parent "${CONFIRMED_ORG}" \
  --repo '<existing-custom-repo>' \
  --file '<assembly-file.yaml>' \
  --dry-run
```

A non-zero dry-run exit can mean changes would be applied; inspect output rather than treating it as an ordinary failure.

Capture reports and logs for both repositories using current help-derived flags:

```bash
chainctl images repos build list \
  --parent "${CONFIRMED_ORG}" \
  --repo '<custom-repo>' \
  -o json

chainctl images repos build logs \
  --parent "${CONFIRMED_ORG}" \
  --repo '<custom-repo>' \
  --build-id '<build-id>'
```

Save sanitized reports and logs. Record source repository/digest, custom repository, build ID and time, output index or manifest digest, package configuration, entitlement/synchronization expiration when applicable, and relevant retry history. Actual reports are authoritative.

## 7. Build a digest-pinned multi-stage image

Use immutable Custom Assembly outputs:

```dockerfile
# syntax=docker/dockerfile:1.7

ARG BUILDER_IMAGE=cgr.dev/<confirmed-org>/<builder-repo>@sha256:<builder-index-digest>
ARG RUNTIME_IMAGE=cgr.dev/<confirmed-org>/<runtime-repo>@sha256:<runtime-index-digest>

FROM ${BUILDER_IMAGE} AS build
USER root
SHELL ["/bin/bash", "-o", "pipefail", "-o", "errexit", "-o", "nounset", "-c"]

ARG SOURCE_COMMIT=<full-commit>
# Fetch or copy only the verified source and inputs.
# Build into /opt/<project> or another controlled prefix.
# Assert the resulting version/revision before leaving this stage.

FROM ${RUNTIME_IMAGE} AS runtime
USER root
# Create the required non-root account and writable directories.
# Copy only built application artifacts and runtime assets.
COPY --from=build /opt/<project> /opt/<project>
USER <non-root-uid>
ENTRYPOINT ["/usr/bin/dumb-init", "--", "/entrypoint"]
```

Adapt the copied artifact:

| Ecosystem | Typical runtime payload |
|---|---|
| Python | virtual environment or controlled prefix |
| Go | binary plus required shared libraries and assets |
| Java | JAR/WAR and matching JRE rather than a JDK where possible |
| Node.js | production dependency tree, compiled output, package metadata |
| C/C++ | installed prefix with binaries, libraries, and runtime data |

Check distro-specific assumptions: `apt`/`dpkg`, Debian or Alpine paths, shell availability, dynamic library paths, `LD_PRELOAD`, user-management commands, CA paths, locale generation, writable `/etc/passwd`, and signal/exec behavior. Make the smallest explainable compatibility change.

Reproducibility statements must distinguish:

- digest-pinned images and full commits: immutable identities;
- checksummed downloads: content-pinned inputs;
- hashed lockfiles: stronger dependency reproduction;
- version-only constraints: not byte-for-byte reproducible; and
- assembly YAML with unversioned package names: can resolve to newer builds, so pin the successful output digest.

## 8. Validate each architecture

Build each requested platform separately for local testing:

```bash
docker buildx build --platform linux/amd64 \
  --file Dockerfile.chainguard \
  --tag '<project>:<version>-chainguard-amd64' --load .

docker buildx build --platform linux/arm64 \
  --file Dockerfile.chainguard \
  --tag '<project>:<version>-chainguard-arm64' --load .
```

Publishing a registry image or multi-architecture index is a separate external mutation. Require an explicit registry destination and authorization.

Validate these layers on every architecture:

1. **Identity/configuration:** app and runtime versions, architecture, labels, digest pins, non-root user, entrypoint, command, port, environment, and workdir.
2. **Package boundary:** list runtime APKs, verify required libraries, run `ldd` or equivalent, and prove representative compilers/headers/build tools are absent.
3. **Runtime behavior:** default entrypoint, signals, writable paths, configured/arbitrary UID behavior, TLS, timezone/locale, health behavior, and read-only-root operation when required.
4. **Application smoke:** initialize configuration, load a required plugin/provider, serve a health endpoint, or process representative input.
5. **Disposable service test:** run migrations or initialization against a suitable disposable database/queue/service. SQLite or an in-memory substitute is insufficient when production behavior differs.
6. **Production-shaped acceptance:** exercise the actual database/version, executor/workers, queue, secrets backend, object store, required extensions, persistent volumes, Kubernetes/Helm deployment, proxies/private CAs, readiness/liveness, startup, upgrade, and rollback behavior that the user relies on.

Native runners provide stronger platform evidence than emulation. Record emulated tests accurately. Record skipped tests and, when known, the owner who will run them.

## 9. Scan and retain supply-chain evidence

Scan every architecture with the same scanner and database snapshot where possible:

```bash
grype 'docker:<project>:<version>-chainguard-amd64'
grype 'docker:<project>:<version>-chainguard-arm64'
```

Record scan date, image digest/config ID, architecture, scanner version, vulnerability database schema/build time, findings by severity and ecosystem, fix availability, and critical installed/fixed versions.

Separate ownership:

- updated entitled APKs can address OS-package findings;
- rebuilding the base cannot fix frozen Python, Java, JavaScript, Go, or application code;
- scanner-recommended upgrades are not automatically compatible with the pin; and
- an EOL exception needs a named risk owner, compensating controls, and a migration or retirement plan.

If comparing with an upstream image, use the same scanner database and comparable architectures. Say “findings” and count unique advisory IDs separately; do not collapse CVE and GHSA aliases into a false CVE count.

For registry outputs, preserve available SBOMs, SLSA provenance, apko configuration attestations, vulnerability/VEX attestations, and signatures. Custom Assembly uses per-organization signer identities; derive the expected identity from real attestations or reports.

## 10. Artifact and evidence contract

A full implementation normally produces:

```text
Dockerfile.chainguard
<chosen-builder-file.yaml>
<chosen-runtime-file.yaml>
CHAINGUARD_BUILD.md
SECURITY_SCAN_REPORT.md
evidence/
  source-identity.txt
  checksums.txt
  chainctl-version.txt
  builder-build-report.json
  builder-build.log
  runtime-build-report.json
  runtime-build.log
  image-inspect-amd64.json
  image-inspect-arm64.json
  smoke-tests.txt
  integration-tests.txt
  package-inventory-amd64.txt
  package-inventory-arm64.txt
  grype-amd64.json
  grype-arm64.json
  attestations/
```

Adapt names to the project and agreed architectures. Keep tags identical across build commands, tests, scans, and documentation. Do not commit credentials, tokens, personal/regulated data, customer input, or unsanitized configuration.

`CHAINGUARD_BUILD.md` should document the pin rationale, exact source/input identities, assembly source/output digests, entitlement/expiration behavior, build and test commands, required production acceptance, reproducibility limits, and residual risk.

## Definition of done

- [ ] Target tag/commit is explicit, immutable, and verified as far as upstream evidence allows.
- [ ] External inputs, submodules, constraints, and patches are pinned/checksummed where possible.
- [ ] Runtime compatibility and redistribution obligations are documented.
- [ ] Builder/runtime package choices have written justification and aligned ABI families.
- [ ] The user selected both assembly filenames and directly confirmed the organization.
- [ ] New custom repositories were created with `--save-as`; source repositories were not modified.
- [ ] Factory reports, logs, build IDs, source digests, and output digests were saved.
- [ ] Dockerfile pins both assembly outputs and fails on source/application identity mismatch.
- [ ] Every requested architecture builds and passes identity/import/configuration smoke tests.
- [ ] Runtime contains required libraries and excludes the development toolchain.
- [ ] Disposable initialization/migration passes where applicable.
- [ ] Required production integrations pass or have explicit gaps and owners.
- [ ] Every architecture has a scan from a recorded scanner/database.
- [ ] OS findings are separated from application/language findings.
- [ ] SBOM, provenance, signatures, and attestations are retained or absence is documented.
- [ ] Residual EOL risk and migration/retirement ownership are explicit.
- [ ] Generated files are tracked or handed off with exact paths.

## Common failure diagnosis

| Symptom | Likely cause | Response |
|---|---|---|
| `repo.create` denied | role lacks create capability | obtain an authorized role; do not modify the base |
| Factory build nears one hour | solve failure or timeout | retrieve the build log and reduce/correct the package set |
| Header or `pkg-config` module missing | builder development package absent | locate an entitled package and justify it |
| Runtime import misses `.so` | runtime package absent or ABI mismatch | inspect dynamic links and align package families |
| New package fails on old source image | libc/ABI incompatibility | choose compatible entitled content or escalate |
| One architecture fails | platform-specific artifact/build path | build and test platforms independently |
| Version command passes, integration fails | smoke test did not exercise services | add production-shaped service/plugin tests |
| Scan still has many findings | frozen language/application graph | separate ownership and document residual risk |
| Rebuild changes contents | floating package or tag resolution | pin successful output digests and inventory results |
| Docs and CLI disagree | version drift | use current CLI help and record `chainctl` version |

## Airflow 2.7.3 lessons that informed this playbook

The reference case pinned Apache Airflow tag `2.7.3` to peeled commit `f1243537838516b8bb8156130bc001595bfbeb01` and a corrected Python 3.11 constraints commit. It used separate builder/runtime assemblies from one entitled source digest, verified source and constraints independently, retained the official extras set, and made only one narrow Debian-path compatibility change in the entrypoint.

Important general lessons:

- A local Wolfi/apko prototype can help discover dependencies but is not a substitute for entitled Chainguard OS Custom Assembly output.
- Development and runtime LDAP/other native ABI families must match.
- Distro-specific preload and library paths must be verified rather than copied.
- Digest pins preserve selected assembly output even when YAML package names float.
- Removing toolchains does not shrink or repair a broad historical language dependency graph.
- Clean OS findings do not make an old application supported or vulnerability-free.
- Exact image tags must remain consistent across docs, tests, and scan evidence.
- Local tests do not replace customer-specific database, executor, workflow, provider, secrets, or Kubernetes acceptance.

## Current authoritative references

- [Custom Assembly overview](https://edu.chainguard.dev/chainguard/containers/features/ca-docs/custom-assembly/)
- [Custom Assembly with chainctl](https://edu.chainguard.dev/chainguard/containers/features/ca-docs/custom-assembly-chainctl/)
- [Custom Assembly FAQ](https://edu.chainguard.dev/chainguard/containers/features/ca-docs/faq/)
- [Chainguard image directory](https://images.chainguard.dev/)
- [Chainguard Privacy Notice](https://www.chainguard.dev/legal/privacy-notice)

Reconcile all links with the installed CLI's current help before acting.
