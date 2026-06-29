# Contributing to Company Brain

Thanks for considering a contribution. Company Brain is infrastructure that agent systems depend on in production — our review bar is high, but the process below is designed to get good changes merged quickly and keep low-effort or risky ones from slipping through.

## Table of Contents

- [Before You Start](#before-you-start)
- [Ways to Contribute](#ways-to-contribute)
- [RFC Process](#rfc-process-required-for-non-trivial-changes)
- [Development Setup](#development-setup)
- [Branch & Commit Conventions](#branch--commit-conventions)
- [Pull Request Requirements](#pull-request-requirements)
- [Code Review Standards](#code-review-standards)
- [Testing Requirements](#testing-requirements)
- [Security-Sensitive Contributions](#security-sensitive-contributions)
- [Publishing Capabilities to the Registry](#publishing-capabilities-to-the-registry)
- [Release Process](#release-process)
- [Maintainers & Governance](#maintainers--governance)
- [Code of Conduct](#code-of-conduct)

---

## Before You Start

- Read the [README](README.md) to understand the resolver/lockfile/registry model — most rejected PRs misunderstand a core invariant (determinism of resolution, signed provenance, scope enforcement).
- Check open issues and the [public roadmap board](#) to avoid duplicate work.
- For anything bigger than a bug fix, open an issue *before* writing code. We'd rather redirect effort early than decline a finished PR.
- All contributors must sign our [Contributor License Agreement](#) (CLA) — the CI bot will prompt you on your first PR.

---

## Ways to Contribute

| Type | Where |
|---|---|
| Bug reports | GitHub Issues, `bug` template |
| Feature requests | GitHub Issues, `feature` template, or an RFC if non-trivial |
| Core engine changes (resolver, policy engine, signing) | RFC required |
| Runtime adapters (new framework support) | Issue + PR, RFC optional unless it changes adapter interface |
| Documentation | PR directly, no RFC needed |
| New public registry capabilities (skills/prompts/MCP bindings) | See [Publishing Capabilities](#publishing-capabilities-to-the-registry) |
| Security reports | **Do not open a public issue** — see [`SECURITY.md`](SECURITY.md) |

---

## RFC Process (required for non-trivial changes)

Non-trivial = anything that changes the manifest schema, lockfile format, resolution semantics, policy engine behavior, signing/provenance format, or any public CLI/SDK interface.

1. Open an issue using the `RFC` template, covering: problem statement, proposed change, backward-compatibility impact, and alternatives considered.
2. A maintainer labels it `rfc:under-review`. Discussion happens in the issue thread for a minimum of 5 business days.
3. Outcome is one of: `accepted` (you or a maintainer may proceed with implementation), `accepted-with-changes`, or `declined` (with rationale).
4. Implementation PRs reference the RFC issue number in the description.

This exists because resolver and lockfile semantics are a contract every downstream agent deployment relies on — silent behavior changes here are the failure mode we most want to avoid.

---

## Development Setup

```bash
git clone https://github.com/companybrain/companybrain.git
cd companybrain
make dev-setup     # installs toolchain, pre-commit hooks, local registry stub
make test          # full unit + integration suite
make test-fast     # unit tests only, for inner-loop iteration
```

Requirements: Go ≥ 1.21 (core CLI/resolver), Python ≥ 3.9 and Node ≥ 18 (SDK test matrices), Docker (integration tests spin up a local registry container).

Run a single package's tests:

```bash
go test ./resolver/... -run TestSemverConflict -v
```

---

## Branch & Commit Conventions

- Branch naming: `feat/<short-desc>`, `fix/<short-desc>`, `docs/<short-desc>`, `rfc/<issue-number>-<short-desc>`.
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `feat:`, `fix:`, `docs:`, `refactor:`, `test:`, `chore:`. This drives automated changelog generation — non-conforming commits will be squashed/rewritten at merge.
- **All commits must be signed** (`git commit -S`). Unsigned commits will fail CI provenance checks — this isn't optional, since the project's own supply-chain integrity model (`brain audit`, signed capabilities) only means something if the tooling that builds it holds itself to the same standard.

---

## Pull Request Requirements

Every PR must:

1. Reference a linked issue (or RFC, for non-trivial changes).
2. Include tests covering the change. New resolver/policy logic requires both unit tests and an integration test against the local registry stub.
3. Pass `make ci-verify` locally before requesting review (mirrors what CI runs: lint, unit tests, integration tests, `brain ci verify` self-check, license header check).
4. Update relevant documentation in the same PR — docs lag is treated as an incomplete PR, not a follow-up.
5. Keep diffs scoped. Mixing a refactor with a behavior change will get a request to split.
6. Fill out the PR template completely, including a "Risk" section for anything touching resolution, policy, or signing.

PRs that don't meet these will be marked `needs-work` rather than reviewed line-by-line — please get the checklist green first so reviewer time goes to substance, not process gaps.

---

## Code Review Standards

- One maintainer approval is required for documentation and adapter-level changes.
- **Two** maintainer approvals are required for changes to the resolver, policy engine, lockfile format, or signing/provenance logic.
- Reviewers evaluate against: correctness, backward compatibility, determinism of resolution output, and whether the change could weaken any security guarantee (signature verification, scope enforcement, audit trail completeness) even incidentally.
- Maintainers may request a design discussion in the issue/RFC thread rather than reviewing a PR in isolation if the change has broader implications than the diff suggests.

---

## Testing Requirements

| Layer | Requirement |
|---|---|
| Unit tests | Required for all new logic; target >90% coverage on resolver/policy packages specifically (`make coverage`) |
| Integration tests | Required for anything touching install/resolve/publish flows; run against `make local-registry` |
| Determinism tests | Resolver changes must include a test asserting identical lockfile output across repeated runs with the same manifest + registry state |
| Adapter conformance | New/changed runtime adapters must pass the shared adapter conformance suite in `test/conformance/` |
| Eval regression | Changes to prompt-handling or capability execution paths should be run against `evals/core-suite.yaml` before merge |

---

## Security-Sensitive Contributions

Changes touching signature verification, scope/policy enforcement, the audit feed, or anything in `pkg/security/` require:

- A maintainer with security sign-off authority as one of the two required approvers.
- A written threat-model note in the PR description: what could go wrong if this is bypassed or misconfigured, and how the change is tested against that.

Do not include real credentials, tokens, or production registry URLs in tests or fixtures — use the local registry stub.

---

## Publishing Capabilities to the Registry

Contributing a *capability* (skill, prompt, tool spec, MCP server binding) to the public registry is a separate process from contributing to this repo:

```bash
brain publish skill:my-new-skill@1.0.0 --registry public --dry-run
```

Public registry submissions go through automated scanning (vulnerability/prompt-injection patterns, license check, scope-declaration completeness) before becoming installable. See [`docs/registry-publishing.md`](docs/registry-publishing.md) for the full guide and trust-score mechanics.

---

## Release Process

- Releases follow semver and are cut from `main` by a maintainer.
- Changelogs are generated from Conventional Commit history and hand-reviewed before publish.
- Breaking changes to the manifest/lockfile schema require a migration guide in `docs/migrations/` as part of the release PR, and a deprecation window of at least one minor version before removal.

---

## Maintainers & Governance

Current maintainers are listed in [`MAINTAINERS.md`](MAINTAINERS.md). Maintainer status is granted by existing maintainer consensus based on sustained, high-quality contribution — there's no fixed contribution count threshold; it's a judgment call based on track record and demonstrated understanding of the resolver/security model.

---

## Code of Conduct

All contributions are governed by our [Code of Conduct](CODE_OF_CONDUCT.md). Participation in this project means agreeing to abide by it.

---

<p align="center">
  <sub>Questions not covered here? Ask in the contributor Slack or open a `question`-labeled issue.</sub>
</p>
