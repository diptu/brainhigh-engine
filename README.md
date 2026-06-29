
# brainhigh-engine

**The dependency manager for AI agents.**

brainhigh-engine is a versioned, auditable registry and runtime for agent capabilities — skills, prompts, tool definitions, and MCP servers — declared as dependencies, resolved deterministically, and shipped the way you'd ship any other package. Think `pip` / `npm`, but the artifact isn't code, it's *agent behavior*.

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](#)
[![License](https://img.shields.io/badge/license-Apache%202.0-blue)](LICENSE)
[![Release](https://img.shields.io/badge/release-v0.1.0-orange)](#)
[![Security](https://img.shields.io/badge/security-SOC%202%20Type%20II%20(in%20progress)-lightgrey)](#security)
[![Slack](https://img.shields.io/badge/community-Slack-4A154B)](#)

---

## Table of Contents

- [Why brainhigh-engine](#why-company-brain)
- [Core Concepts](#core-concepts)
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [Installation](#installation)
- [The `brain.yaml` Manifest](#the-brainyaml-manifest)
- [CLI Reference](#cli-reference)
- [SDKs](#sdks)
- [Registry](#registry)
- [Versioning & Resolution](#versioning--resolution)
- [Security Model](#security-model)
- [Observability](#observability)
- [Enterprise Deployment](#enterprise-deployment)
- [Configuration Reference](#configuration-reference)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Support & SLAs](#support--slas)
- [License](#license)

---

## Why brainhigh-engine

Every AI-native company is independently re-solving the same problem: agents accumulate skills, prompts, tool bindings, and MCP server connections with no version control, no dependency graph, no rollback path, and no audit trail. A prompt change in one agent silently breaks three downstream workflows. A skill gets duplicated four times across teams with subtle drift. Nobody can answer "what exactly was this agent capable of on March 3rd?"

brainhigh-engine treats agent capabilities as first-class, versioned artifacts:

- **Reproducible** — pin exact versions of skills, prompts, and MCP servers; lockfiles guarantee identical agent behavior across environments.
- **Composable** — declare capabilities as dependencies with semantic versioning, transitive resolution, and conflict detection.
- **Auditable** — every capability change is signed, timestamped, and attributable to a commit, actor, and approval chain.
- **Governable** — org-level policy enforcement (allow-lists, capability scopes, data-access boundaries) applied at install time, not at runtime postmortem.
- **Portable** — the same manifest runs your agent locally, in CI, and in production, regardless of underlying model provider or orchestration framework (LangGraph, AutoGen, custom runtimes).

If your organization runs more than a handful of agents in production, brainhigh-engine is the layer that keeps them from becoming unmanageable.

---

## Core Concepts

| Concept | Description |
|---|---|
| **Capability** | A unit of agent behavior: a skill, a prompt template, a tool/function spec, or an MCP server binding. |
| **Manifest (`brain.yaml`)** | Declares which capabilities an agent depends on, with version constraints. |
| **Lockfile (`brain.lock`)** | Fully resolved, hashed, pinned dependency graph — the source of truth for what actually gets installed. |
| **Registry** | Where capabilities are published, versioned, and discovered (public registry or private/self-hosted). |
| **Resolver** | Computes a consistent dependency graph from manifests, respecting semver constraints and policy rules. |
| **Runtime Adapter** | Translates resolved capabilities into framework-native objects (LangGraph nodes, OpenAI tool specs, MCP client configs, etc.). |
| **Policy** | Org-defined rules constraining which capabilities, scopes, or data-access patterns are installable. |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         brainhigh-engine CLI                       │
│              (init · add · install · audit · publish)           │
└───────────────────────────┬───────────────────────────────────┘
                              │
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
       ┌────────────┐  ┌─────────────┐  ┌──────────────┐
       │  Resolver  │  │   Policy    │  │   Signing /  │
       │  Engine    │  │   Engine    │  │   Provenance │
       └─────┬──────┘  └──────┬──────┘  └───────┬──────┘
              │                │                  │
              └────────────────┼──────────────────┘
                                ▼
                      ┌───────────────────┐
                      │  brain.lock        │
                      │ (resolved graph)   │
                      └─────────┬─────────┘
                                ▼
                    ┌────────────────────────┐
                    │   Runtime Adapters      │
                    │  LangGraph │ AutoGen │   │
                    │  MCP Client │ Custom     │
                    └─────────────────────────┘
                                ▼
                    ┌────────────────────────┐
                    │   Your Agents in Prod   │
                    └────────────────────────┘

         ▲
         │  publish / pull
         ▼
┌─────────────────────────────────────────────────────────┐
│   Registry (Public)  ──or──  Private Registry (Self-host) │
│   Skills · Prompts · Tool Specs · MCP Server Bindings      │
└─────────────────────────────────────────────────────────┘
```

---

## Quick Start

```bash
# Install the CLI
curl -fsSL https://get.companybrain.dev | sh

# Initialize a new agent project
brain init my-support-agent
cd my-support-agent

# Add capabilities as dependencies
brain add skill:ticket-triage@^2.1.0
brain add mcp:zendesk-connector@1.4.2
brain add prompt:escalation-policy@~1.0

# Resolve and install
brain install

# Verify the resolved graph
brain tree
```

`brain install` produces a `brain.lock` file. Commit it. It is the contract that guarantees your agent behaves identically in dev, staging, and prod.

---

## Installation

### CLI

| Platform | Command |
|---|---|
| macOS / Linux | `curl -fsSL https://get.companybrain.dev \| sh` |
| Homebrew | `brew install companybrain/tap/brain` |
| Windows (PowerShell) | `iwr https://get.companybrain.dev/install.ps1 -useb \| iex` |
| Docker | `docker pull companybrain/cli:latest` |

### Language SDKs

```bash
pip install companybrain          # Python
npm install @companybrain/sdk     # Node / TypeScript
go get github.com/companybrain/go-sdk
```

### Minimum Requirements

- Git ≥ 2.30
- Network access to your registry endpoint (public or private)
- One of: Python ≥ 3.9, Node ≥ 18, Go ≥ 1.21 (for SDK usage)

---

## The `brain.yaml` Manifest

```yaml
name: support-agent
version: 0.4.0

runtime:
  framework: langgraph
  model_router: multi-model   # multi-model | single-model
  default_provider: anthropic

dependencies:
  skills:
    ticket-triage: "^2.1.0"
    sentiment-analysis: "~1.3.0"
  prompts:
    escalation-policy: "1.0.2"
    refund-decision-tree: "^3.0.0"
  mcp_servers:
    zendesk-connector: "1.4.2"
    internal-kb-search: "git+https://github.com/acme/kb-mcp#v2.0.0"
  tools:
    create_refund: "internal/tools/refunds@1.0.0"

policy:
  scope_allowlist:
    - read:customer_pii        # explicitly granted
  scope_denylist:
    - write:billing_system     # explicitly forbidden, overrides any dependency request
  require_approval_for:
    - mcp_servers               # any new MCP server requires reviewer sign-off

eval:
  suite: "evals/support-agent-v3.yaml"
  min_pass_rate: 0.92
  gate_on_failure: true
```

---

## CLI Reference

| Command | Description |
|---|---|
| `brain init <name>` | Scaffold a new agent project with manifest and lockfile. |
| `brain add <type>:<name>@<version>` | Add a capability dependency to the manifest. |
| `brain remove <type>:<name>` | Remove a dependency. |
| `brain install` | Resolve manifest into a lockfile and materialize capabilities locally. |
| `brain update [name]` | Update one or all dependencies within allowed semver ranges. |
| `brain tree` | Print the resolved dependency graph. |
| `brain audit` | Scan installed capabilities for known vulnerabilities, deprecated versions, or policy violations. |
| `brain publish` | Publish a capability (skill/prompt/tool/MCP binding) to a registry. |
| `brain diff <ref1> <ref2>` | Show capability-level diff between two lockfile revisions — critical for change review. |
| `brain rollback <version>` | Revert an agent to a previously locked, known-good capability set. |
| `brain eval` | Run the declared eval suite against the currently resolved capability graph. |
| `brain ci verify` | CI-friendly command: fails the build if `brain.lock` doesn't match `brain.yaml`, or if policy/eval gates fail. |

---

## SDKs

**Python**
```python
from companybrain import Brain

brain = Brain.from_lockfile("brain.lock")
agent_tools = brain.resolve_tools(framework="langgraph")
prompt = brain.get_prompt("escalation-policy")
```

**TypeScript**
```typescript
import { Brain } from "@companybrain/sdk";

const brain = await Brain.fromLockfile("brain.lock");
const tools = brain.resolveTools({ framework: "vercel-ai-sdk" });
```

Runtime adapters ship for **LangGraph**, **AutoGen**, **CrewAI**, **OpenAI Assistants API**, **MCP clients**, and a generic adapter interface for custom orchestration layers.

---

## Registry

- **Public Registry** (`registry.companybrain.dev`) — open, community-published capabilities. Subject to automated security scanning and a public trust score (downloads, audit history, maintainer reputation).
- **Private Registry** — self-hosted or managed, scoped to your org. Supports SSO-gated publish/pull, internal-only namespaces, and mirroring of approved public packages.

```bash
brain registry add internal https://brain.acme.internal --auth sso
brain config set default-registry internal
```

Every published capability carries:
- Semantic version
- Signed provenance (publisher identity, build origin, commit SHA)
- SBOM-equivalent manifest of its own transitive dependencies
- Declared scopes / data-access requirements

---

## Versioning & Resolution

brainhigh-engine uses standard semver (`MAJOR.MINOR.PATCH`) for all capability types.

- **MAJOR** — breaking change to prompt contract, tool signature, or MCP server interface.
- **MINOR** — backward-compatible capability additions.
- **PATCH** — bug fixes, prompt wording corrections that don't change behavior contracts.

Resolution is deterministic: given the same `brain.yaml` and registry state, `brain install` always produces an identical `brain.lock`. Conflicting transitive version constraints fail the resolution with an explicit, human-readable conflict report rather than silently picking a version (no agent should run on an unintentionally resolved dependency graph).

---

## Security Model

- **Signed capabilities** — every published artifact is cryptographically signed (Sigstore-compatible); `brain install` verifies signatures by default and refuses unsigned capabilities unless explicitly allowed.
- **Scope enforcement** — capabilities declare required scopes (e.g., `read:customer_pii`, `write:billing_system`). Org policy can allow-list or deny-list scopes at install time — denylist always wins.
- **Approval gates** — `require_approval_for` in the manifest forces human sign-off for sensitive capability classes (e.g., any new MCP server, anything with `write:` scopes) before `brain install` will proceed in CI.
- **Supply-chain auditing** — `brain audit` checks installed capabilities against a continuously updated vulnerability/deprecation feed, analogous to `npm audit` / `pip-audit`, but scoped to prompt-injection patterns, jailbreak-prone prompt templates, and known-malicious MCP servers, in addition to standard CVEs.
- **Immutable provenance** — lockfiles record the exact hash, source, signer, and timestamp of every resolved capability, enabling full reconstruction of "what could this agent do, and where did that capability come from" for any historical point in time.

See [`SECURITY.md`](SECURITY.md) for vulnerability disclosure policy and our PGP key.

---

## Observability

brainhigh-engine emits structured events for every capability resolution, install, and invocation:

```json
{
  "event": "capability.invoked",
  "agent": "support-agent",
  "capability": "skill:ticket-triage@2.1.3",
  "lockfile_hash": "sha256:9f2a...",
  "trace_id": "a1b2c3",
  "timestamp": "2026-06-29T10:14:02Z"
}
```

Native exporters: **OpenTelemetry**, **Datadog**, **Honeycomb**. This enables "which exact capability version caused this production incident" queries against your existing observability stack, rather than a bespoke brainhigh-engine dashboard you have to context-switch into.

---

## Enterprise Deployment

| Capability | OSS | Enterprise |
|---|---|---|
| Public registry access | ✅ | ✅ |
| Private/self-hosted registry | ❌ | ✅ |
| SSO / SCIM provisioning | ❌ | ✅ |
| Policy engine (scope allow/deny, approval gates) | Basic | Advanced (org-wide, role-based) |
| Audit log retention | 7 days | Unlimited, exportable to SIEM |
| SLA-backed support | ❌ | ✅ ([see below](#support--slas)) |
| On-prem / VPC deployment | ❌ | ✅ |
| Compliance attestations (SOC 2, HIPAA-ready config) | ❌ | ✅ |

Enterprise deployments run the Registry and Resolver as services inside your VPC, with no capability metadata or lockfile content leaving your network boundary unless explicitly published to a federated/public registry.

```bash
helm repo add companybrain https://charts.companybrain.dev
helm install company-brain companybrain/company-brain \
  --namespace agent-infra \
  --set registry.mode=private \
  --set registry.persistence.storageClass=ssd-retain
```

Full Helm chart values and Terraform modules: [`deploy/`](deploy/).

---

## Configuration Reference

| Variable | Description | Default |
|---|---|---|
| `BRAIN_REGISTRY_URL` | Default registry endpoint | `https://registry.companybrain.dev` |
| `BRAIN_AUTH_TOKEN` | Auth token for private registry access | — |
| `BRAIN_POLICY_FILE` | Path to org-wide policy overlay | `./brain-policy.yaml` |
| `BRAIN_LOCKFILE_STRICT` | Fail install if lockfile is stale vs. manifest | `true` |
| `BRAIN_TELEMETRY_ENDPOINT` | OTel collector endpoint | unset (telemetry disabled) |
| `BRAIN_CACHE_DIR` | Local capability cache location | `~/.cache/companybrain` |

---

## Roadmap

- [x] Core resolver + lockfile format (v0)
- [x] LangGraph / AutoGen / MCP runtime adapters
- [ ] Capability-level eval gating in CI (`brain eval --gate`)
- [ ] Cross-org federated registries with trust scoring
- [ ] Automatic prompt-drift detection (semantic diffing across patch versions)
- [ ] Native Kubernetes operator for capability hot-reload without agent restart
- [ ] Marketplace for vetted, commercially licensed enterprise skills

Track progress on the [public roadmap board](#).

---

## Contributing

We welcome issues, RFCs, and pull requests.

1. Read [`CONTRIBUTING.md`](CONTRIBUTING.md) and [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).
2. For non-trivial changes, open an RFC issue before submitting a PR.
3. All PRs require passing CI (`brain ci verify`), one maintainer approval, and signed commits.

```bash
git clone https://github.com/companybrain/companybrain.git
cd companybrain
make dev-setup
make test
```

---

## Support & SLAs

| Tier | Channel | Response Time |
|---|---|---|
| Community | GitHub Issues / Discord | Best effort |
| Pro | Email support | < 1 business day |
| Enterprise | Dedicated Slack + named TAM | < 4 hours (P1), < 1 hour for production-down |

Enterprise customers: see [`docs/enterprise-support.md`](docs/enterprise-support.md) for escalation paths and on-call coverage windows.

---

## License

Core CLI, resolver, and SDKs are licensed under [Apache 2.0](LICENSE). The hosted public Registry and Enterprise control plane are proprietary and licensed separately — see [`docs/licensing.md`](docs/licensing.md).

---

<p align="center">
  <sub>Built for the agents your company is already running. Built before they get away from you.</sub>
</p>
