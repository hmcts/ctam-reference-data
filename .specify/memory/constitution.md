# CTAM Reference Data Constitution

This constitution governs the `ctam-reference-data` repository, which fetches reference data
(including JO and MRD data) from upstream APIs, persists the raw payloads in a blob store, and
then reads, transforms and persists that data into a PostgreSQL database. The Core Principles
are derived from the HMCTS Programming Principles
(https://hmcts.github.io/standards/principles/programming.html) and applied to this pipeline.

## Core Principles

### I. YAGNI - You Aren't Gonna Need It

Code MUST be written only for requirements that exist in an approved specification. Speculative
abstractions, configuration switches, unused API clients, unused transformation paths and
"future-proofing" tables or columns MUST NOT be added. Where a design choice is made to allow a
hypothetical future need, the pull request MUST justify it against a documented, scheduled
requirement; otherwise it MUST be removed.

Rationale: every unused path is code that must be tested, secured and maintained without
delivering value, and it obscures the real data flow.

### II. KISS - Keep It Simple, Stupid

The simplest design that satisfies the specification MUST be preferred. Each stage of the
pipeline (fetch, store, transform, load) MUST be understandable in isolation. Frameworks,
libraries and patterns MUST NOT be introduced unless they remove more complexity than they add.
Any complexity beyond the straightforward solution MUST be recorded in the plan's Complexity
Tracking table with the reason the simpler alternative was rejected.

Rationale: reference data pipelines fail in operation, not in design; simple code is faster to
diagnose and safer to change.

### III. Security Baked-In, Not Bolted On

Security MUST be designed into every change from the start, in line with the NCSC Cloud
Security Principles. Specifically:

- Secrets (API keys, connection strings, storage keys) MUST NOT be committed to the repository
  and MUST be sourced from Azure Key Vault or managed identity at runtime.
- All network calls to upstream APIs, blob storage and PostgreSQL MUST use TLS.
- Access to blob containers and database roles MUST follow least privilege; the pipeline
  identity MUST hold only the permissions it needs.
- Dependencies MUST be scanned for known vulnerabilities in CI, and HIGH or CRITICAL findings
  MUST be resolved or formally accepted before merge.
- Personal or sensitive data MUST NOT be written to logs.

Rationale: retrofitting security to a data pipeline that has already ingested data is costly
and often impossible.

### IV. Transparency - Work in the Open

All source code, specifications, plans, tasks, pipeline definitions and this constitution MUST
live in this public GitHub repository. Design decisions MUST be recorded in the relevant
`specs/` artefacts or pull request description, not in private channels. The only material
excluded from the repository is secrets and personal data. Branch names, commit messages and
pull request titles MUST describe the change plainly.

Rationale: HMCTS works in the open; visible history is how future maintainers understand why
the data looks the way it does.

### V. Peer Review - Nothing Reaches Production Without Review

Every change to `main` MUST arrive via a pull request approved by at least one reviewer who is
not the author. Direct pushes to `main` MUST be blocked by branch protection. Reviewers MUST
verify compliance with this constitution, that automated tests exist for the change, and that
CI is green. Emergency changes are not exempt: they MUST still be reviewed, and may be reviewed
after deployment only when the incident record says so.

Rationale: review is the primary quality and security control for a pipeline that writes
authoritative reference data.

### VI. Egoless Programming - You Are Not Your Code

Review feedback MUST address the code, not the person, and MUST be received on the same basis.
Authors MUST NOT block or dismiss review comments without a technical reason recorded in the
thread. Any team member MAY change any part of the codebase; there is no individual ownership
of modules. Defects are treated as learning for the team, and blameless language MUST be used
in incident and retrospective records.

Rationale: shared ownership and open critique produce better code and a healthier team than
defensiveness does.

### VII. Domain Driven Design - Design Around the Domain

The codebase MUST be organised around the reference data domain, not around technical layers.
Domain concepts (for example JO and MRD data sets, their source systems, and the published
reference tables) MUST be modelled explicitly with names taken from the business vocabulary
and used consistently in code, database schema and documentation (a ubiquitous language).
Bounded contexts MUST be kept separate: raw ingestion models MUST NOT leak into the
transformed, published models, and translation between them MUST happen in a named
transformation layer.

Rationale: reference data is meaningful only in its business context; code that mirrors the
domain is easier to validate with domain experts.

### VIII. SOLID

Object-oriented code in this repository MUST follow the SOLID principles:

- Single Responsibility: each class has one reason to change. Fetching, storing,
  transforming and loading MUST be separate components.
- Open/Closed: new data sources or transformations MUST be added by adding components, not by
  modifying existing working ones with conditionals.
- Liskov Substitution: implementations of a shared interface (for example an API client or
  a blob repository) MUST be interchangeable without callers changing behaviour.
- Interface Segregation: interfaces MUST be small and specific to their consumer.
- Dependency Inversion: components MUST depend on abstractions (interfaces) for external
  systems so that they can be tested without live APIs, storage or databases.

Rationale: SOLID keeps the pipeline testable and lets new data sets be added with low risk.

### IX. Composition Over Inheritance

Behaviour MUST be composed from small collaborating objects rather than inherited from deep
class hierarchies. Inheritance MAY be used only for genuine "is-a" relationships and MUST NOT
exceed one level of concrete inheritance without written justification in the pull request.
Shared behaviour across fetchers, transformers or loaders MUST be extracted into collaborators
that are injected, not into abstract base classes.

Rationale: composition keeps components independently testable and avoids fragile base class
changes rippling through the pipeline.

### X. Law of Demeter - Talk to Friends, Not Strangers

An object MUST only call methods on itself, its fields, its parameters, and objects it creates.
Chained navigation through other objects' internals (`a.getB().getC().doThing()`) MUST NOT be
used; instead the immediate collaborator MUST expose the needed operation. Data transfer
objects and records used purely to carry data across boundaries are exempt.

Rationale: limiting knowledge of other objects' structure keeps coupling low and makes
refactoring of upstream API or database models safe.

## Additional Constraints

### Technology stack

- Language and framework: Java 25 and Spring Boot 4.1.1, built with Gradle.
- Raw payload storage: Azure Blob Storage.
- Transformed data storage: Azure Database for PostgreSQL.
- Database schema changes MUST be applied through versioned Liquibase changelogs
  checked into the repository; manual schema changes are prohibited.
- Secrets and configuration MUST be supplied through Azure Key Vault or environment
  configuration, never through committed files.

### Data pipeline rules

- Raw API responses MUST be persisted to blob storage unchanged before any transformation
  occurs, and MUST be treated as immutable once written.
- Blob objects MUST be named so that source system, data set, and fetch timestamp are
  recoverable from the name or metadata.
- Every fetch, transform and load stage MUST be idempotent: re-running a stage with the same
  input MUST produce the same result without duplicating rows or blobs.
- Every row written to PostgreSQL MUST be traceable to the blob it was derived from and the
  pipeline run that wrote it.
- Transformations MUST be pure functions of their input blob(s) and MUST NOT call upstream
  APIs; failed transformations MUST fail the run visibly rather than write partial data
  silently.
- Pipeline runs MUST emit structured logs and metrics (start, end, counts read/written, and
  failures) sufficient to diagnose a failed run without re-executing it.

## Development Workflow

- Work MUST follow the Spec Kit flow: specification, plan and tasks in `specs/` before
  implementation, with each feature developed on its own branch.
- Every pull request MUST include automated tests for the change. Unit tests MUST cover
  transformation logic; integration tests MUST cover each external boundary (API client, blob
  repository, database repository) using test doubles or containers. Test-first development
  is encouraged but not mandated.
- CI MUST run build, tests, static analysis and dependency vulnerability scanning on every pull
  request, and all checks MUST pass before merge.
- Pull requests MUST be small enough to review in one sitting and MUST state which constitution
  principles were considered where a design decision was non-obvious.
- Reviewers MUST confirm the Constitution Check in the feature plan has been satisfied.

## Governance

This constitution supersedes all other development practices in this repository. Where a
specification, plan, task or code conflicts with it, the constitution wins and the conflicting
artefact MUST be corrected.

Amendments MUST be proposed via a pull request that edits this file, states the reason for the
change, and identifies any existing code or artefacts that become non-compliant along with a
plan to bring them into compliance. Amendments require the same peer review as code.

Versioning follows semantic versioning:

- MAJOR: a principle is removed or redefined in a backward-incompatible way.
- MINOR: a principle or section is added, or existing guidance is materially expanded.
- PATCH: clarifications, wording and typo fixes with no change of meaning.

Every pull request review MUST include a check for compliance with this constitution.
Complexity that departs from principles I, II, VIII, IX or X MUST be justified in the plan's
Complexity Tracking table. The constitution MUST be reviewed at least once every six months
and the review outcome recorded in the pull request or a `specs/` note, even if no change is
made.

**Version**: 1.0.0 | **Ratified**: 2026-09-18 | **Last Amended**: 2026-09-18
