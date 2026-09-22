# CTAM Reference Data Constitution

This constitution governs the `ctam-reference-data` repository, which fetches reference data
(including JO and MRD data) from upstream APIs, persists the raw payloads in a blob store, and
then reads, transforms and persists that data into a PostgreSQL database. The Core Principles
are derived from the HMCTS Programming Principles
(https://hmcts.github.io/standards/principles/programming.html) and applied to this pipeline.
The Additional Constraints adopt the conventions of the HMCTS Spring Boot template and the
Cloud Native Platform (CNP), as exemplified by `hmcts/service-api-marketplace`.

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
Libraries listed under Approved baseline libraries are pre-approved and need no per-change
justification. Any complexity beyond the straightforward solution MUST be recorded in the
plan's Complexity Tracking table with the reason the simpler alternative was rejected.

Rationale: reference data pipelines fail in operation, not in design; simple code is faster to
diagnose and safer to change.

### III. Security Baked-In, Not Bolted On

Security MUST be designed into every change from the start, in line with the NCSC Cloud
Security Principles. Specifically:

- Secrets (API keys, connection strings, storage keys) MUST NOT be committed to the repository
  and MUST be sourced from Azure Key Vault via workload identity at runtime. The only exemption
  is default credentials for the local Docker Compose environment (for example a local
  PostgreSQL username and password); these MUST be usable only against local containers and
  MUST NOT be valid for any deployed environment.
- All network calls to upstream APIs, blob storage and PostgreSQL MUST use TLS.
- Access to blob containers and database roles MUST follow least privilege; the pipeline
  identity MUST hold only the permissions it needs.
- Dependencies MUST be scanned for known vulnerabilities in CI using the HMCTS Java Gradle
  plugin's OWASP dependency check. HIGH or CRITICAL findings MUST be resolved or formally
  accepted before merge. Formal acceptance means an entry in the OWASP suppressions file that
  states the reason, the affected package and a review date; suppressions without a reason
  MUST be rejected in review.
- Dependency updates MUST be kept current via Renovate using the HMCTS shared configuration.
- Personal or sensitive data MUST NOT be written to logs. Caller-supplied values MUST be
  encoded before logging to prevent log injection.

Rationale: retrofitting security to a data pipeline that has already ingested data is costly
and often impossible.

### IV. Transparency - Work in the Open

All source code, specifications, plans, tasks, Helm chart, Terraform for project-specific
resources, Jenkins pipeline files and this constitution MUST live in this public GitHub
repository. Configuration that HMCTS shared tooling requires to live elsewhere is permitted
only in the following sanctioned repositories:

- `hmcts/cnp-flux-config` for Flux HelmRelease definitions, per-environment value overlays,
  image policies and workload identity service accounts.
- `hmcts/cnp-jenkins-config` for team configuration, Terraform approval whitelists and
  deployment controls.
- The HMCTS shared-infrastructure project for shared Azure resources (see Infrastructure and
  deployment).

Every change to those repositories on behalf of this service MUST be referenced from a pull
request, `specs/` artefact or living document in this repository so the full picture is
recoverable from here. Design decisions MUST be recorded in the relevant `specs/` artefacts or
pull request description, not in private channels. The only material excluded from all
repositories is secrets and personal data. Branch names, commit messages and pull request
titles MUST describe the change plainly.

Rationale: HMCTS works in the open; visible history is how future maintainers understand why
the data looks the way it does.

### V. Peer Review - Nothing Reaches Production Without Review

Every change to `main` MUST arrive via a pull request approved by at least one reviewer who is
not the author. Direct pushes to `main` MUST be blocked by branch protection, and the Jenkins
pull request check MUST be a required status check. Reviewers MUST verify compliance with this
constitution, that automated tests exist for the change, and that CI is green. Emergency
changes are not exempt: they MUST still be reviewed, and may be reviewed after deployment only
when the incident record says so.

The single exception is Renovate dependency updates. PATCH and MINOR version bumps raised by
Renovate MAY be automerged once all CI checks pass. MAJOR version bumps MUST be reviewed by a
human like any other change.

Rationale: review is the primary quality and security control for a pipeline that writes
authoritative reference data. Automerged patch and minor bumps keep the dependency baseline
current at low risk because CI, vulnerability scanning and tests still gate them.

### VI. Egoless Programming - You Are Not Your Code

Review feedback MUST address the code, not the person, and MUST be received on the same basis.
Authors MUST NOT block or dismiss review comments without a technical reason recorded in the
thread. Any team member MAY change any part of the codebase; there is no individual ownership
of modules. Defects are treated as learning for the team, and blameless language MUST be used
in incident and retrospective records.

Rationale: shared ownership and open critique produce better code and a healthier team than
defensiveness does.

### VII. Domain Driven Design - Design Around the Domain

The root package MUST be `uk.gov.hmcts.ctam.refdata`, reflecting the CNP product and
component, and every class MUST sit in a technical sub-package beneath it (for example
`controllers`, `services`, `repository`, `entity`, `domain`, `mappers`, `config`, `scheduling`),
following the HMCTS Spring Boot template layout. Within that layout the domain MUST still be
modelled explicitly: domain concepts (for example JO and MRD data sets, their source systems,
and the published reference tables) MUST be named from the business vocabulary and used
consistently in code, database schema and documentation (a ubiquitous language). Bounded
contexts MUST be kept separate in the type system even though they share technical packages:
raw ingestion models MUST NOT leak into the transformed, published models, and translation
between them MUST happen in a named transformation layer (for example a mapper per data set).
Class and table names MUST make the bounded context they belong to unambiguous.

Rationale: the template layout keeps the repository familiar to other HMCTS teams, while a
ubiquitous language and separated models keep the data meaningful in its business context and
easy to validate with domain experts.

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
  systems so that they can be tested without live APIs, storage or databases. Dependencies
  MUST be supplied by constructor injection.

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
objects, records and JPA entities used purely to carry data across boundaries are exempt.

Rationale: limiting knowledge of other objects' structure keeps coupling low and makes
refactoring of upstream API or database models safe.

## Additional Constraints

### Runtime model

- The service MUST be an always-on Spring Boot application deployed through the CNP `java`
  Helm chart. It MUST listen on port 8080 and expose the actuator health endpoint at the root
  management path so that CNP startup and readiness probes succeed.
- Pipeline stages MUST be invoked by the Spring Boot internal scheduler. Kubernetes CronJobs
  or external schedulers MUST NOT be used.
- The health endpoint MUST report database connectivity in its readiness group.

### Technology stack

- Language and framework: Java 25 and Spring Boot 4.1.1, built with Gradle using the checked-in
  wrapper. Compiler warnings MUST be treated as errors.
- Raw payload storage: Azure Blob Storage.
- Transformed data storage: Azure Database for PostgreSQL Flexible Server.
- Persistence for the load stage MUST use Spring Data JPA.
- Database schema changes MUST be applied through versioned Flyway SQL migrations under
  `src/main/resources/db/migration`, checked into the repository; manual schema changes are
  prohibited.
- Secrets and configuration MUST be supplied through Azure Key Vault mounted via workload
  identity and Spring config tree import, never through committed files (subject to the
  local-only exemption in Principle III).
- Container images MUST be built from the HMCTS distroless Java 25 base image.

### Approved baseline libraries

The following are pre-approved under Principle II and need no per-change justification:
Lombok, MapStruct, Spring Data JPA, Testcontainers, the HMCTS Java Gradle plugin, and the
Spring Boot starters for web, actuator, validation, data-jpa and flyway. Any other library MUST
be justified in the plan's Complexity Tracking table.

### Naming and ownership

- CNP product is `ctam` and component is `refdata`. All derived names MUST follow the CNP
  `{product}-{component}` scheme: Helm chart and release `ctam-refdata`, container image
  `hmctsprod.azurecr.io/ctam/refdata`, Kubernetes namespace `ctam`, Key Vault `ctam-{env}`,
  managed identity `ctam-{env}-mi`.
- The GitHub team `@hmcts/ctam` MUST be the code owner in `CODEOWNERS` and the Backstage owner
  group in `catalog-info.yaml`.
- The default branch is `main`. Pipeline and workflow files copied from HMCTS templates MUST
  be adjusted from `master` to `main`.

### Infrastructure and deployment

- Shared Azure infrastructure (resource groups, Key Vault, managed identities, PostgreSQL
  Flexible Server) MUST be provisioned by Terraform in the HMCTS shared-infrastructure project.
- Project-specific Azure resources (for example blob storage containers) MUST be provisioned by
  Terraform in the `infrastructure/` folder at the root of this repository.
- Manual changes to Azure resources are prohibited. Where a one-off manual step is unavoidable
  it MUST be recorded in a living document under `specs/` with the ticket reference, the
  command or portal action taken, and the follow-up to automate it.
- Deployment MUST use the HMCTS Jenkins CNP pipeline (`Jenkinsfile_CNP` with the shared
  Infrastructure library) with Flux GitOps for AKS. The service MUST NOT be deployed by any
  other route.

### Observability and CI

Logging, metrics, tracing and CI tooling MUST follow HMCTS shared tooling. The specific
libraries, dashboards and scan jobs are not pinned in this version and MUST be pinned in a
subsequent amendment once aligned with the platform team. Until then the Data pipeline rules
on structured logs and metrics apply as stated.

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
  implementation, with each feature developed on its own branch. Living documents such as
  onboarding plans and runbooks MUST also live under `specs/`, not at the repository root.
- Every pull request MUST include automated tests for the change. Unit tests MUST cover
  transformation logic; integration tests MUST cover each external boundary (API client, blob
  repository, database repository) using test doubles or Testcontainers. Test-first
  development is encouraged but not mandated.
- Test source sets MUST follow the HMCTS template: `test` for unit tests, `integrationTest`
  for integration tests, and `smokeTest` for a post-deployment check that calls the exposed
  health endpoint. The `functionalTest` source set is optional for this service.
- Line coverage measured by JaCoCo across unit and integration tests MUST be at least 70%,
  and the SonarQube quality gate MUST pass. Coverage exclusions MUST be limited to the
  application entry point, JPA entities and exception classes.
- CI MUST run build, unit and integration tests, Checkstyle, and OWASP dependency scanning on
  every pull request via the HMCTS Java Gradle plugin, and all checks MUST pass before merge.
- Pull requests MUST be small enough to review in one sitting. The pull request template MUST
  include a Constitution Check section, and authors MUST state which principles were considered
  where a design decision was non-obvious.
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

**Version**: 1.1.0 | **Ratified**: 2026-09-18 | **Last Amended**: 2026-09-22
