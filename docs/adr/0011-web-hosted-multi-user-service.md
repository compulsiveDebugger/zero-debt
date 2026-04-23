# ADR-0011: Web-Hosted Multi-User Service (Not a GitHub App, Not CLI-First)

**Status:** Accepted

## Context

Early framing of zero-debt as "GitHub-native" was ambiguous. It could mean:

1. A **GitHub App** — installed on an organization, acting via GitHub's installation tokens, permission model, and webhook-first lifecycle.
2. A **self-hosted service** that *uses* the GitHub API but runs under the customer's infrastructure and has its own user model.
3. A **CLI tool** that wraps `gh` and issues commits/PRs from a developer's laptop.

Business requirements crystallized:

- It is an **in-house corporate tool** — users log in via their corporate identity (eventually ZTIAP).
- Users **connect specific repositories** they own or have access to, configure per-repo settings, and run phases from a UI.
- Long-running runs must survive browser closure.
- Audit, cost attribution, and authorization are required — all of which imply a user model *inside* zero-debt, not delegated to GitHub.

## Decision

zero-debt is a **self-hosted, multi-user web application**. Specifically:

- Users authenticate to zero-debt (DB-based v1; ZTIAP v2 — see [ADR-0013](0013-pluggable-authentication.md)).
- Users connect GitHub repositories via **Personal Access Token** (v1) or **GitHub OAuth** (later). Their PAT is encrypted at rest with a deployment-level KEK.
- zero-debt is **not** a GitHub App. No installation flow. No GitHub-issued installation tokens.
- A CLI exists for internal operations and testing but is not the primary entry point. The primary interface is the Angular UI backed by a FastAPI API.

## Consequences

**Positive**

- Enterprise deployment is straightforward — a container stack running under the customer's VPC, their Postgres, their Mongo.
- Auth integrates cleanly with ZTIAP because identity is ours to own, not delegated to GitHub.
- No dependence on GitHub App marketplace approval or permission model constraints.
- Per-user / per-repo authorization, audit, and cost attribution are first-class.

**Negative**

- Users must manage their own GitHub credentials (PAT). A GitHub App would avoid this friction. Mitigated by OAuth flow as a later enhancement.
- We carry the weight of being an identity provider (however thin). `DbAuthProvider` must meet enterprise security standards from v1 because it ships first.

**Non-negotiable**

- No GitHub App installation flow in v1 or v2. If a customer wants that experience, it's a future product.
