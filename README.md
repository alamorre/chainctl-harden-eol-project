# chainctl-harden-eol-project

A Codex skill for rebuilding an unmaintained GitHub project—or an exact older tag or commit—on a Chainguard OS layer without silently changing the application identity.

The workflow guides an agent through:

- immutable source and release-input verification;
- inspection of the pinned revision's real build behavior;
- separate Chainguard Custom Assembly builder and runtime images;
- digest-pinned multi-stage application builds;
- per-architecture smoke, integration, package-boundary, and security checks; and
- an evidence bundle that separates OS remediation from residual EOL application risk.

It was derived from a successful Apache Airflow 2.7.3 rebuild, but it intentionally requires package selection, runtime behavior, entrypoint changes, and acceptance tests to be rediscovered for every project.

## Install

Clone the repository into your Codex skills directory:

```bash
git clone https://github.com/alamorre/chainctl-harden-eol-project.git \
  ~/.codex/skills/chainctl-harden-eol-project
```

Restart or reload Codex if needed, then invoke it with:

```text
$chainctl-harden-eol-project
```

Example request:

```text
Use $chainctl-harden-eol-project to rebuild release 1.8.4 from
https://github.com/example/project for linux/amd64 and linux/arm64.
Preserve the application and dependency versions, and stop before publishing.
```

## Prerequisites

- Git and a container build environment
- `chainctl` and access to an eligible Chainguard Production Containers organization
- Explicit user confirmation of the target Chainguard organization before any organization-scoped CLI operation
- Project-specific acceptance criteria and any required disposable integration services

Custom Assembly is additive and entitlement-bound. It modernizes the OS package layer; it does not make frozen upstream code supported, vulnerability-free, or production-ready without project-specific validation.

See [SKILL.md](SKILL.md) for the agent instructions and [the implementation playbook](references/eol-custom-assembly-playbook.md) for the detailed workflow.
