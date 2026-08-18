# ADR 0002: Backend modernization

## Status

Proposed

## Date

2026-08-18

## Context

ShaDo is a single Node.js process that serves the built React application and a MongoDB-backed checklist API. The API is implemented in `checklist.service/server.js`, `checklist.service/controllers/checklistController.js`, and `checklist.service/models/checklistModel.js`.

The current implementation is intentionally small, but its platform and operational assumptions are now risky:

| Area | Current state and finding |
| --- | --- |
| Runtime and dependencies | `checklist.service/package.json` declares no Node engine and uses Express `^4.17.1`, MongoDB driver `^3.5.9`, Mongoose `^5.13.20`, Jest `^26.1.0`, and Supertest `^4.0.2`. The service lockfile is npm lockfile version 1 (`checklist.service/package-lock.json`). These major versions are several generations behind supported releases. The direct `mongodb` dependency is not imported by the service; all database access uses Mongoose. |
| API design | `checklist.service/server.js` exposes seven unversioned routes below `/api/checklists`: create, retrieve, replace/update and delete a checklist, plus create, replace/update and delete an embedded item. The React client calls those routes directly in `checklist.ui/src/service.js`, so preserving their response shapes while clients migrate is important. Successful creation and update return `200`, not `201`/a documented contract. There is no health, readiness, or API version endpoint. |
| Data model and persistence | `checklist.service/models/checklistModel.js` defines only optional `title`, `description`, `lastUpdated`, and unbounded embedded `listItems` fields. It has no required fields, length constraints, schema validation, indexes, document ownership, optimistic concurrency, or automatic timestamps. `update` passes a spread of the request body to `findOneAndUpdate` (`controllers/checklistController.js`), allowing replacement-style omission of fields and relying on implicit Mongoose casting. Item mutations read then save whole documents for add/delete, creating avoidable lost-update races; `lastUpdated` is application-managed. |
| Sharing and authorization | The README describes link-based collaborative editing (`README.md`), but every operation, including delete, is publicly reachable by its MongoDB `_id`. There is no user identity, checklist owner, per-share capability, expiry, revocation, role separation, or audit record. An ObjectId in a URL is functioning as an undocumented bearer credential and grants write/delete access. |
| Input and errors | `server.js` accepts JSON and URL-encoded bodies without an explicit size limit. Controllers do not validate ObjectIds or request DTOs, do not normalize error responses, and turn invalid IDs, casting failures, and database outages into generic `500` text responses. `delete` returns `204` even when no checklist was deleted, while deleting an absent item from an existing checklist also returns `204`. There is no centralized async error boundary or `404` handler. |
| Security | CORS is both enabled with `cors()` and manually widened with `Access-Control-Allow-Origin: *` in `server.js`. There are no security headers, rate/abuse controls, request-size policy, authentication, authorization, CSRF strategy for an authenticated future, dependency audit gate, or sensitive-field redaction policy. |
| Observability and resilience | Startup and all controller failures use `console.log`; the connection error is logged but does not stop startup (`server.js`). There is no structured logging, request ID, error reporting, metrics, tracing, readiness check, graceful shutdown, connection lifecycle management, timeout policy, or signal handling. |
| Configuration and secrets | The only service configuration is `DB_URL` and `PORT` (`checklist.service/server.js`, `checklist.service/index.js`). `DB_URL` is not checked before connecting, and there is no configuration schema, environment example, secret-management guidance, or runtime environment selection. `.gitignore` ignores generated `public/` and local dependencies but provides no `.env.example`. |
| Test coverage | The sole service suite, `checklist.service/test/server.test.js`, mocks Mongoose calls and tests selected happy/404 controller paths. It does not exercise a real database, connection failure, validation, malformed IDs, all error branches, concurrency, authorization/sharing, CORS/security headers, or deployment health. The root `package.json` test command intentionally fails, so repository-level test automation cannot validate the service. |
| Delivery and deployment | Root `build.sh` runs `npm install` twice rather than deterministic `npm ci`, writes to `checklist.service/public`, and uses `mkdir` without `-p`. The only process declaration is lower-case `procfile` with `web: npm start`; Heroku recognizes `Procfile` (capital P), so that declaration is not reliable. There is no pinned runtime, container image, multi-stage build, health check, CI workflow, release/migration step, or deployment configuration. |

## Decision

Modernize the backend incrementally while retaining MongoDB and the existing checklist route contract during the transition.

1. Establish a supported baseline: run the service on Node.js 24 LTS, declare the supported Node and npm versions in `engines` and a version file, use npm lockfile version 3 with `npm ci`, and upgrade to supported Express 5, Mongoose 8, test tooling, and transitive dependencies. Remove the unused direct `mongodb` dependency after confirming Mongoose is the only client. Add Dependabot or equivalent automated dependency updates and an audit policy.
2. Convert the service to TypeScript in a dedicated `src/` layout, retaining Express as the HTTP framework and MongoDB as the database. Use explicit request/response DTOs and runtime validation (Zod or an equivalent schema library) at the route boundary. Configure JSON and URL-encoded body limits, validate identifiers before database access, use RFC 9457-style JSON problem responses, and add centralized not-found and error middleware.
3. Define `/api/v1` as the documented API surface. Keep the existing `/api/checklists` handlers as a compatibility layer until the React client uses v1, then retire them on a published schedule. Use correct status codes going forward (`201` for creation, `204` only for confirmed idempotent deletion) and publish an OpenAPI document and generated/request-contract tests.
4. Evolve the checklist document deliberately. Require and bound titles and items, use Mongoose timestamps, add an integer version for optimistic concurrency, and add only indexes justified by access patterns. Use atomic `$push`, `$pull`, and positional updates for item changes. Bound document growth and establish a migration path to a separate item collection if checklist size or write contention exceeds MongoDB document limits.
5. Replace ObjectId-as-write-capability sharing with an explicit model. Introduce users (or an external identity provider), checklist ownership, and share records/capability tokens. Use cryptographically random, separately stored share-token hashes with scopes (`read` or `edit`), expiry, revocation, and optional rotation; never expose the token hash. Enforce authorization on every route. Continue to support existing public links only through a time-boxed compatibility path and communicate the required migration.
6. Make production operation observable and fail-safe. Validate configuration at startup, source secrets from the platform secret store, use structured redacted logs with correlation IDs, and expose liveness/readiness endpoints. Treat a failed MongoDB connection as unavailable, set database/server timeouts, handle `SIGTERM` by draining HTTP traffic and closing Mongoose, and emit metrics and error telemetry through the hosting platform's supported integration.
7. Secure the HTTP edge with an explicit CORS allowlist, Helmet or equivalent headers, rate limits sized for public sharing, request-size limits, TLS-only deployment, and abuse monitoring. Revisit CSRF protections if browser-cookie authentication is selected.
8. Package and deploy deterministically. Replace `build.sh` with lockfile-based, reproducible scripts; split UI and service build stages in a multi-stage Dockerfile (or use an explicitly supported Node buildpack); use a correctly cased `Procfile` only if retaining Heroku; and add CI for install, type-check, tests, build, dependency audit, and image/deployment smoke checks.

## Consequences

### Positive

- The service gains a supported runtime and dependency supply chain, reproducible builds, and deployable health signals.
- Validated, versioned API contracts make client changes and operational failures predictable.
- Ownership and revocable scoped sharing turn an incidental identifier into an intentional access-control model.
- Atomic updates, validation, timestamps, and concurrency controls improve data integrity under simultaneous collaborators.
- Structured telemetry and graceful shutdown shorten incident diagnosis and reduce deploy-related data/availability risk.

### Negative

- TypeScript, validation, authentication, observability, CI, and container/build configuration add code and operational complexity to a currently simple app.
- Stronger sharing controls require data migration, client UX decisions, and communication to users with existing links.
- Rate limiting and CORS restrictions can disrupt undocumented integrations unless rollout is measured and allowlisted.
- Dependency upgrades may expose legacy Mongoose and Express behavior differences; they must not be combined with untested schema/API redesign in one release.

### Risks and mitigations

- **Public-link breakage:** inventory active documents, offer a compatibility window, and issue/redeem migration tokens before disabling ObjectId write access.
- **MongoDB migration defects:** back up data, run idempotent migrations with dry-run/report modes, verify counts and samples, and retain a rollback plan.
- **API behavior regressions:** characterize the current client flow with contract tests before introducing `/api/v1`; shadow or canary the new handlers.
- **Operational cost:** start with the hosting platform's logging and metrics integrations, and set retention/alert thresholds before adding a separate observability vendor.

## Alternatives Considered

### Rewrite immediately using a different backend framework and database

Rejected. Replacing Express, MongoDB, and the data model at once maximizes risk without solving the immediate security and supportability gaps faster. MongoDB's document model remains suitable for a checklist while its lifecycle, validation, and access model are corrected.

### Keep anonymous ObjectId links and add only framework upgrades

Rejected. The primary security issue is not merely an old package: any holder of an enumerable application identifier can mutate or delete a checklist. Upgrades without an explicit sharing capability model leave that design unchanged.

### Use a managed backend/BaaS for authentication and data

Deferred. A managed identity service may be selected during the ownership/share design, but a wholesale BaaS migration would couple API, authorization, data migration, and hosting changes. The proposed API boundary permits that decision later.

### Keep JavaScript and add runtime validation only

Viable as a short-term stepping stone, but not selected as the target. Runtime validation is mandatory either way; TypeScript additionally makes controller, model, and API-contract refactors safer once the service grows.

## Phased Migration Plan

### Phase 0: Baseline and characterization

1. Inventory production Node, MongoDB, hosting, active checklist counts, document sizes, and consumers of `/api/checklists`.
2. Capture the existing client interactions from `checklist.ui/src/service.js` as contract tests, including failure behavior.
3. Back up MongoDB and record a restore exercise. Add CI that runs the existing service tests independently from the intentionally failing root `test` script.

### Phase 1: Safe platform upgrade

1. Pin Node.js 24 LTS and npm, regenerate lockfiles, replace `npm install` with `npm ci`, and correct the Heroku `Procfile` or introduce the chosen container deployment.
2. Upgrade Express, Mongoose, Jest, and Supertest in small, tested changes; remove the unused direct MongoDB driver.
3. Add startup configuration validation, graceful shutdown, health/readiness routes, structured logging, and CI build/test/audit jobs.

### Phase 2: API hardening without client breakage

1. Add centralized errors, request identifiers, body limits, CORS allowlisting, security headers, rate limiting, ObjectId validation, and DTO validation to existing handlers.
2. Correct data-model constraints and timestamps using a backward-compatible migration. Replace read-modify-write item operations with atomic operators and introduce optimistic concurrency.
3. Add integration tests against an ephemeral MongoDB instance for validation, persistence, errors, malformed identifiers, and concurrent writes.

### Phase 3: Versioned API and authorization

1. Publish OpenAPI for `/api/v1`, implement v1 handlers, and migrate the React client from the unversioned routes.
2. Introduce identity, checklist ownership, and hashed scoped share tokens with expiry and revocation. Add authorization, audit events, and security tests.
3. Migrate existing documents and links through a monitored compatibility period, with feature flags and rollback criteria.

### Phase 4: Cutover and operational maturity

1. Monitor v1 adoption, error rates, latency, rate-limit rejections, and MongoDB document growth; canary then fully route clients to v1.
2. Deprecate and remove unauthenticated unversioned write endpoints after the published deadline and invalidate legacy write capabilities.
3. Review production dashboards, alerts, backup/restore evidence, dependency posture, and capacity limits quarterly.
