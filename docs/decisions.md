# Decisions (ADR-style)

Short records of decisions that shaped the portal. Each one is grounded in the production code base; the code itself is not published.

## 1. Re-check authorization in every service, not only at the edge

**Context.** Next.js offers a request proxy (middleware) that can block routes before rendering. It is convenient, but it only sees the URL and the session, not which project a query will read. A client must never read another client's project.

**Decision.** The proxy does a route-level check (signed-in, role allowed for this area). Every service and route handler then calls `requirePermission` and `requireProjectAccess` against the `ProjectMember` table before reading or writing. Repositories contain no authorization logic and are never called directly from routes.

**Consequences.** Some checks run twice per request. In exchange, adding a new route or a new consumer (for example, the in-app assistant's tools) cannot bypass project isolation by accident: the service it calls already enforces it.

## 2. One public endpoint, written as an allowlist with a double gate

**Context.** A separate public map application needs a subset of vessel operation plans. Everything else in the portal requires authentication.

**Decision.** A dedicated service file with no auth imports, so reviewers see immediately that the openness is intentional. The repository query filters on two independent flags (`publicationStatus = PUBLISHED` and `publicVisibility = PUBLISHED_PUBLIC`) and `select`s each returned field by name. The response is always an array, never a singleton.

**Consequences.** Publishing to the public map is a separate, explicit action from publishing to the client. New database columns never reach the public endpoint without a code change. The response shape is pinned by tests in both applications after a flat-vs-nested contract mismatch once broke the consumer without any error on the producer side.

## 3. E-mail through a deduplicated outbox

**Context.** Notifications are triggered from request handlers that can be retried by the browser or the platform. Sending directly from the handler risks duplicates, and development environments must never e-mail real people.

**Decision.** Store each message first in an `EmailOutbox` table with a unique `dedupeKey`, then attempt delivery through Microsoft Graph. If no provider is configured, the row is marked `SKIPPED`.

**Consequences.** At most one e-mail per logical event, an audit trail of what was sent, and a safe default. Delivery is still attempted inline rather than by a background worker, which is acceptable at current volume.

## 4. Fall back to local data only for known infrastructure failures

**Context.** The client-facing pages existed before the database did, and had to keep working while the database was being rolled out. Silently showing fixture data in production, however, would be misleading.

**Decision.** A `withDatabaseFallback` wrapper used only on client read paths. It falls back only on recognised connection, authentication, timeout or missing-schema errors, only outside production, and is never used by administrative mutations. Any other error propagates.

**Consequences.** Demos keep working without a database, and real bugs are not masked. The cost is maintaining a small list of error signatures.

## 5. Manual zip deployment of the standalone build (for now)

**Context.** The App Service runs a plain `node server.js` with no server-side build configured, and the running app should not depend on installing packages at deploy time.

**Decision.** Build locally with Next.js `output: "standalone"`, copy static assets into the bundle with a post-build script, and upload the bundle as a zip deployment.

**Consequences.** Predictable artefacts and no build step on the server. The process depends on one person's machine and discipline; moving it into a CI pipeline with an approval gate is the main open item.
