# DmarcAnalyzerApp: structure, norms and API

Oct 19, 2026 · @kletel multimédia

## Overview

DmarcAnalyzerApp (v0.14.0) is a self-hosted, agency-first DMARC analyzer: one workspace monitors aggregate (RUA) and TLS-RPT reports for many clients across many domains, and ships as one container image plus PostgreSQL.

| Layer | Technology |
| --- | --- |
| Backend | .NET 10 (`net10.0`), ASP.NET Core minimal APIs grouped with Carter 10 |
| Persistence | PostgreSQL 18, EF Core 10 + Npgsql (raw SQL where EF cannot express it) |
| Ingestion | MailKit (IMAP, POP3), AWS S3 SDK (bucket sources), HTTP push, `DmarcRua` parser, SharpCompress for zip/gzip payloads |
| DNS | DnsClient (DMARC, MX, MTA-STS, TLS-RPT lookups) |
| Auth | Local passwords or OIDC (`Microsoft.AspNetCore.Authentication.OpenIdConnect`), machine tokens |
| Observability | OpenTelemetry (ASP.NET Core, Npgsql, HttpClient, runtime), off until an OTLP endpoint is set |
| Frontend | React 19, React Router 7, Vite 8, TypeScript 5.9 strict, Tailwind CSS 3 |
| Tests | xUnit (InMemory EF) + Testcontainers PostgreSQL; Vitest for the web app |

One image runs in the mode `APP_MODE` selects: `api` (REST API + built SPA from `wwwroot`), `worker` (ingestion host), `all` (both in one process, the shipped default), `migrate` (apply migrations, exit) or `mta-sts` (internet-facing host serving only the MTA-STS policy routes and health probes). Any other value fails startup.

## Repository layout

Everything builds from one solution (`DmarcAnalyzerApp.slnx`) and one `Dockerfile`; shared MSBuild settings (locked restore in CI and the image) live in `src/Directory.Build.props`.

| Path | Holds |
| --- | --- |
| `src/api/Program.cs` | Single entrypoint; branches on `APP_MODE` and builds the web host, worker host, migrator or MTA-STS host |
| `src/api/Modules/` | 20 Carter modules, one per resource area; thin HTTP mapping only |
| `src/api/Application/<Area>/` | Service layer: Analytics, ApiCredentials, Audit, Auth, Backup, Clients, Common, Domains, Hosting, Ingestion, MtaSts, Notifications, Observability, ReportSources, Reports, Retention, Security, Users |
| `src/api/Contracts/<Area>/` | Request records bound from JSON bodies (`CreateXRequest`, `UpdateXRequest`); response DTOs sit beside their service as `*Dtos.cs` |
| `src/api/Data/` | `DmarcAnalyzerDbContext`, 30 entities in `Entities/`, EF `Migrations/`, design-time factory, connection-string resolver |
| `src/api/Middleware/` | `MachineAuthMiddleware`, `SessionAuthMiddleware`, `RoleAuthorizationMiddleware` |
| `src/api/Workers/` | `QueueWorkerService`, `WorkerOptions`, `WorkerSingleInstanceLock` |
| `src/api.tests/` | xUnit unit tests on EF InMemory; `Fixtures/` holds sample reports |
| `src/api.integration.tests/` | Testcontainers PostgreSQL tests for raw SQL, `ON CONFLICT` dedup and transactions |
| `src/web/src/pages/` | One component per console route (Dashboard, Domains, DomainDetail, Threats, Clients, ReportSources, Alerts, Notifications, Users, Audit, Backup, FirstRunImport) |
| `src/web/src/components/` | Shell (`ConsoleLayout`, `AuthProvider`, `LoginPage`), `ui/` and `data/` primitives, feature cards |
| `src/web/src/lib/` | Typed API client (`api.ts`), domain types (`entities.ts`, `analytics.ts`), hooks and pure helpers |
| `deploy/` | Shipped Compose files, Helm chart, Umbrel and CasaOS app-store manifests |
| `docs/planning/`, `docs/ops/` | Architecture, data model, API contract, ADRs 0001-0010, roadmap/backlog/status; operator runbooks |
| `http/api.http` | REST Client request collection |
| `.github/workflows/` | `ci.yml`, `prepare-release.yml` |

## Backend architecture

Requests pass three custom middlewares in a fixed order, then reach a thin Carter handler that calls one `Application/` service and maps its `ServiceResult<T>` onto HTTP.

&#91;embedded content: request path and worker · 7 layers, 1 background loop\]

A request stops at the first middleware that rejects it; the worker reaches the same service layer without going through HTTP.

**Request pipeline** (`Program.cs`, api/all modes):

1. Forwarded headers (only when `Network__*` is on with a non-empty trust list), CORS (dev only), default + static files for the SPA.
2. `UseAuthentication()` only when `Auth:Oidc:Enabled`; it backs the OIDC external-temp scheme, not the app session.
3. `MachineAuthMiddleware` resolves `Authorization: Bearer` machine tokens into `IMachineCallerContext`. It never touches the user context.
4. `UseRateLimiter()` (the report-ingest policy).
5. `SessionAuthMiddleware` resolves the `dmarc_session` cookie into `CurrentUserContext` for every `/api/v1/*` path except the public list (`auth/login`, `register`, `logout`, `setup`, `providers`, `auth/oidc/*`). Missing cookie returns `401 {"error":"not authenticated"}`; a dead one `401 {"error":"session expired or invalid"}`.
6. `RoleAuthorizationMiddleware` reads endpoint metadata and returns `403 {"error":"forbidden"}` when the caller does not qualify.
7. `/health/live`, `/health/ready`, MTA-STS public routes, then `MapCarter()`, then `MapFallbackToFile("index.html")` for SPA routes.

**Module and service split.** Each `ICarterModule` maps routes for one area and injects one service interface (`IClientService`, `IDomainService`, ...) plus `IAuditLog`. Services return `ServiceResult<T>`: a value with 200, or a user-safe error plus the intended status code. The handler turns failures into `Results.Json(new { error }, statusCode)`, successes into `Ok`/`Created`, and records an audit event after every write (`audit.RecordAsync(AuditEvents.ClientCreated, ...)`).

**Worker.** `QueueWorkerService` runs on an interval: it syncs every active polled source (IMAP, POP3, S3) through the transport-agnostic `MailboxSyncService`, then runs periodic passes as they come due: DNS refresh, MTA-STS checks, alerts, digests, database retention, mailbox retention, backup offload. There is no job queue or claim path. `WorkerSingleInstanceLock` takes a Postgres advisory lock so a second worker exits instead of duplicating email and moving checkpoints backwards.

**MTA-STS host.** `APP_MODE=mta-sts` builds a separate minimal app: `/.well-known/mta-sts.txt` (GET/HEAD), `/mta-sts/ask` (Caddy on-demand TLS check) and health probes. No Carter, no SPA fallback, no auth middleware, no migrations.

## Data model and tenancy

`client` is the tenant root: every client-owned row reaches a client directly or through a chain of foreign keys, and a cross-tenant id must answer **404, never 403**, so the API is no existence oracle. Full schema: `docs/planning/data-model.md`.

| Area | Tables (snake\_case via `ToTable`) | Key rules |
| --- | --- | --- |
| Identity and access | `agency_user`, `user_session`, `user_identity`, `user_client_grant`, `api_credential` | `Email`, `CookieId` unique; OIDC identity unique on `(Issuer, Subject)`; grant unique on `(UserId, ClientId)`; role is a string column, no role table |
| Clients and domains | `client`, `domain` | `client.Slug` unique; `domain.Name` globally unique, so one domain belongs to one client; `RetentionMonths` default 27, `LegalHold` skips purging |
| Sources and sync | `report_source`, `mailbox_sync_run` | `report_source.DefaultClientId` decides where unknown domains are auto-created |
| DMARC reports | `dmarc_report_ingest`, `dmarc_report`, `dmarc_report_record`, `..._dkim_auth_result`, `..._spf_auth_result`, `report_ingest_receipt` | Dedup on `(DomainId, ReportId, RangeBeginUtc, RangeEndUtc)` with `ON CONFLICT DO NOTHING`; report data cascades on delete |
| TLS-RPT (RFC 8460) | `smtp_tls_report`, `smtp_tls_report_policy`, `smtp_tls_failure_detail`, `tls_report_ingest` | Dedup on `(OrganizationName, ReportId, RangeBeginUtc, RangeEndUtc)` |
| Notifications | `notification_recipient`, `notification_recipient_client`, `alert_event`, `digest_delivery` | Recipients route per client |
| MTA-STS | `mta_sts_state`, `mta_sts_policy` | Policy hosted per domain |
| Audit and backup | `audit_event`, `backup_stream_state`, `report_backfill_state`, `report_backfill_candidate` |  |

Modelling conventions: one PostgreSQL database; PascalCase columns (EF default, quoted in SQL); business entities (`client`, `domain`, `report_source`) delete with **restrict** so tenant data cannot be orphaned silently. Retention runs daily and measures against the report window end (`RangeEndUtc`), not ingest date.

Tenancy chains: `client ← domain.ClientId ← dmarc_report.DomainId ← dmarc_report_record ← auth results`; `client ← report_source.DefaultClientId ← mailbox_sync_run`; `client ← dmarc_report_ingest.ClientId`; `client ← user_client_grant.ClientId`.

Domain concepts the code depends on:

- **Enforcement status** per domain is derived from the *effective* DMARC policy plus compliance: `enforced` (p=reject), `ramping` (p=quarantine), `spoofing` (unprotected and failing), `monitoring`, `no_data`. A subdomain with no record inherits the organisational domain's `sp=` (else `p=`); `domain.DnsPolicyInheritedFrom` records the source.
- **Analytics windows** anchor to the newest report date, not wall-clock, because data is often backfilled.

## API conventions

The API is JSON over `/api/v1`, authenticated by an HTTP-only session cookie for people and a bearer token for machines, with role checks decided centrally and **deny-by-default**. `docs/planning/api-contract.md` §0 is the authoritative list; its later sections mix in planned endpoints.

**Authentication**

| Caller | Mechanism | Details |
| --- | --- | --- |
| Person, local password | `POST /api/v1/auth/login` sets `dmarc_session` | `HttpOnly`, `SameSite=Lax`, `Secure` follows the request scheme; 12 h idle timeout, 7 day absolute maximum |
| Person, OIDC | `GET /api/v1/auth/oidc/login` → IdP → `/auth/oidc/complete` | Mints the same `dmarc_session` cookie; roles and grants are always decided in the app, never by the IdP (ADR 0007) |
| Machine | `Authorization: Bearer dmarcanalyzer_<tokenId>_<secret>` | Issued by `/api/v1/api-credentials`; only the secret's hash is stored; each credential has a kind (today `ReportIngest`) and a bound report source |

**Authorization tiers** (route metadata read by `RoleAuthorizationMiddleware`):

| Tier | Stamp | Who |
| --- | --- | --- |
| staff | none (default) or `.RequireAgencyStaff()` | `agency_admin`, `agency_analyst` |
| admin | `.RequireAgencyAdmin()` | `agency_admin` only |
| any | `.AllowClientViewer()` | all roles incl. `client_viewer`; the service must scope data via `ICurrentUserContext.CanAccessClient` |
| machine | `.RequireMachineCredential(kind)` | only a bearer credential of that kind; a session, even an admin's, gets 403 |
| public | listed in `SessionAuthMiddleware.PublicPaths` | no session: `auth/login`, `register`, `logout`, `setup`, `providers`, `auth/oidc/*`, `/health/*` |

A machine credential is refused on every endpoint that has not opted in, so a report-ingest token cannot reach the console API.

**Request and response shape**

- Ids are GUIDs, enforced with route constraints (`{id:guid}`, `{domainId:guid}`).
- Request bodies bind to records in `Contracts/<Area>/` (`CreateClientRequest`, `UpdateDomainRequest`); updates use `PATCH`, whole-set replacements use `PUT` (`/users/{id}/grants`, `/notification-recipients/{id}/routing`).
- Filters and windows are query parameters: `days`, `clientId`, `limit`, `offset`, `eventType`. There is no shared page envelope; list endpoints return arrays, and the few paged ones (audit events) take `limit`/`offset`.
- Creates return `201 Created` with a `Location` header; other successes `200`; queued work `202` (report backfill run).
- Errors are a flat `{"error": "<message>"}` with the status from `ServiceResult<T>`: 400 validation, 401 no/invalid session, 403 wrong role or credential, 404 missing **or other tenant's** id, 409 state conflicts, 413 oversized push payload, 429 rate limit.
- Times are ISO-8601 UTC.

The contract doc still describes a nested `{"error":{"code","message","details","traceId"}}` envelope and `page`/`pageSize` paging; neither is implemented. Treat that as design intent, not the current API.

## API endpoint reference

86 routes across 20 Carter modules plus the MTA-STS public routes, taken from the route registrations in `src/api/Modules/*.cs` on 2026-10-19. Paths are relative to `/api/v1` unless they start with `/.well-known`, `/mta-sts` or `/health`. Access tiers are the ones defined under API conventions.

### Auth and system

| Method | Path | Access | Purpose |
| --- | --- | --- | --- |
| POST | `/auth/login` | public | Password login; sets `dmarc_session` |
| POST | `/auth/register` | public | First-run bootstrap only; locked once any user exists |
| POST | `/auth/logout` | public | Ends the session |
| GET | `/auth/setup` | public | `{ requiresBootstrap }`; also the container health probe, since it hits the DB |
| GET | `/auth/providers` | public | Which login front doors are enabled |
| GET | `/auth/oidc/login` | public | Starts the OIDC challenge (`returnUrl`) |
| GET | `/auth/oidc/complete` | public | Finishes OIDC, provisions or links the user, mints the session |
| GET | `/auth/me` | any | Current user, role, granted clients |
| GET | `/system/status` | any | Service name, runtime `mode`, `version`, `revision` |
| GET | `/health/live`, `/health/ready` | public | Liveness; readiness checks DB connectivity (503 when down) |

### Clients, domains and users

| Method | Path | Access | Purpose |
| --- | --- | --- | --- |
| GET | `/clients` | any | List clients (viewers see granted ones only) |
| GET | `/clients/{id}` | any | One client; 404 outside grants |
| POST | `/clients` | admin | Create client; audited |
| PATCH | `/clients/{id}` | admin | Update name, retention, legal hold; audited |
| GET | `/domains` | any | List domains, optional `clientId` |
| GET | `/domains/{id}` | any | One domain |
| POST | `/domains` | admin | Create domain under a client |
| PATCH | `/domains/{id}` | admin | Update or move domain |
| GET | `/users` | admin | List users |
| POST | `/users` | admin | Create user with role |
| PATCH | `/users/{id}` | admin | Change role, name, active state |
| DELETE | `/users/{id}` | admin | Remove user |
| PUT | `/users/{id}/grants` | admin | Replace a viewer's client grants |

### Report sources, credentials and ingestion

| Method | Path | Access | Purpose |
| --- | --- | --- | --- |
| GET | `/report-sources` | staff | List IMAP, POP3, S3 and push sources |
| POST | `/report-sources` | admin | Create source (password encrypted at rest) |
| PATCH | `/report-sources/{id}` | admin | Update source |
| POST | `/report-sources/{id}/sync` | staff | Manual sync now |
| GET | `/mailbox-sync-runs` | staff | Sync history (`reportSourceId`, `limit`, default 50) |
| GET | `/mailbox-health` | staff | Per-source health |
| GET | `/api-credentials` | admin | List machine credentials |
| POST | `/api-credentials` | admin | Issue credential; token returned once |
| POST | `/api-credentials/{id}/revoke` | admin | Revoke credential |
| POST | `/reports` | machine (`ReportIngest`) | Push one report payload; rate-limited; 413 above `Worker:MaxPushedReportRequestBytes`; optional `X-Report-Provenance` |

### Analytics

All take `days` (default 30, clamped to 365), anchor to the newest report, and are `any` unless marked.

| Method | Path | Access | Purpose |
| --- | --- | --- | --- |
| GET | `/analytics/summary` | any | Compliance totals, daily trend, top failing domains and reporters, dispositions, mailbox rollup (staff only) |
| GET | `/analytics/domains` | any | Per-domain compliance, effective policy, enforcement status |
| GET | `/analytics/domains/{domainId}/drilldown` | any | Totals and trend for one domain |
| GET | `/analytics/domains/{domainId}/sources` | any | Per-source-IP aggregation, worst first |
| GET | `/analytics/domains/{domainId}/source-detail` | any | One source (`ip` required): DKIM×SPF combinations, identifiers, reporters, trend |
| GET | `/analytics/domains/{domainId}/enforcement` | any | Next policy step, rationale, `readyToAdvance`, blocking sources |
| GET | `/analytics/domains/{domainId}/report-notes` | any | Parser warnings and errors on the domain's reports, max 100 |
| GET | `/analytics/domains/{domainId}/records` | any | Live DMARC/SPF DNS records parsed per tag, compared with `policy_published` |
| GET | `/analytics/domains/{domainId}/mta-sts` | any | Stored MTA-STS state (record, policy file, MX coverage) |
| POST | `/analytics/domains/{domainId}/mta-sts/recheck` | staff | Live MTA-STS check (DNS + HTTPS), persisted |
| GET | `/analytics/domains/{domainId}/tls-rpt` | any | TLS-RPT sessions, success rate, failure breakdown, live `_smtp._tls` record |
| GET | `/analytics/domains/{domainId}/live-mx` | any | Live MX lookup |
| GET | `/analytics/threats` | any | Fully unauthenticated sources across visible domains (`limit` default 100, max 500) |
| GET | `/analytics/hostnames` | any | Reverse DNS for `ips` (comma-separated, max 100) |

### MTA-STS hosting

| Method | Path | Access | Purpose |
| --- | --- | --- | --- |
| GET | `/domains/{domainId}/mta-sts-policy` | any | Hosted policy plus CNAME/TXT publish instructions |
| PUT | `/domains/{domainId}/mta-sts-policy` | admin | Upsert; policy id bumps only when content changes |
| DELETE | `/domains/{domainId}/mta-sts-policy` | admin | Stop hosting |
| POST | `/clients/{clientId}/mta-sts-policy/apply` | admin | Apply one policy to many domains; per-domain `created`/`updated`/`unchanged` |
| GET, HEAD | `/.well-known/mta-sts.txt` | public | Policy file keyed on `Host`; never redirects |
| GET | `/mta-sts/ask?domain=` | public | Caddy on-demand TLS gate: 200 issue, 403 refuse |

### Alerts, notifications and digests

| Method | Path | Access | Purpose |
| --- | --- | --- | --- |
| GET | `/alerts` | any | Alert history (`days`); client-scoped |
| PATCH | `/alerts/{id}` | staff | Triage to `open`, `acknowledged` or `closed` |
| POST | `/admin/alerts/evaluate` | admin | Evaluate alert rules now |
| GET | `/notification-recipients` | staff | Recipients with `digestDefaultMode` and `clientModes` |
| POST | `/notification-recipients` | admin | Add recipient; modes `rollup`, `separate`, `off` |
| PUT | `/notification-recipients/{id}/routing` | admin | Replace digest routing |
| DELETE | `/notification-recipients/{id}` | admin | Remove recipient |
| POST | `/admin/notifications/test` | admin | Send test email (`to`) |
| GET | `/admin/digest/defaults` | staff | Instance `Digest:*` triggers |
| GET | `/admin/digest/preview` | admin | Render digests for `clientId` or `recipientId` without sending |
| GET | `/admin/digest/preview.html` | admin | One rendered mail as `text/html` |
| POST | `/admin/digest/send` | admin | Send due digests; skips periods already sent |

### Administration, backup and retention

| Method | Path | Access | Purpose |
| --- | --- | --- | --- |
| POST | `/admin/database/migrate` | admin | Apply pending EF migrations |
| GET | `/admin/audit-events` | admin | Audit trail (`days`, `eventType`, `actor`, `clientId`, `limit`, `offset`); read-only |
| GET | `/admin/retention/preview` | admin | What the next purge would delete, per client |
| POST | `/admin/retention/purge` | admin | Run the purge now (`batchSize`) |
| GET | `/admin/mailbox-retention/preview` | admin | Per source: cutoff, eligible messages, suspension reason |
| POST | `/admin/mailbox-retention/purge` | admin | **Irreversible** expunge of report mail past retention |
| GET | `/admin/config/export` | admin | Configuration artifact; 409 without an encryption key unless plaintext is allowed |
| GET | `/admin/config/import/preview` | admin | What an import would change |
| POST | `/admin/config/import/dry-run` | admin | Validate an uploaded artifact without writing |
| POST | `/admin/config/import` | admin | `restore` (empty install only) or additive `merge` |
| GET | `/admin/backup/status` | admin | Offload destination, last success, bucket versioning |
| POST | `/admin/backup/offload` | admin | Run an offload pass now |
| GET | `/admin/report-backfill/preview` | admin | What a backfill from the report-mail archive would find |
| GET | `/admin/report-backfill/status` | admin | Backfill cursor: status, pass, counts, last error |
| POST | `/admin/report-backfill/run` | admin | Queue backfill (202); 409 without archive or while running |

**Drift between code and contract doc:** the code lets `client_viewer` call `GET /clients`, `/clients/{id}`, `/domains` and `/domains/{id}` (contract says staff), and has `DELETE /users/{id}`, `POST /admin/config/import/dry-run` and `GET /analytics/domains/{domainId}/live-mx`, which §0 does not list.

## Frontend structure

The console is a React 19 SPA built by Vite into the API's `wwwroot` and served from the same origin, so it calls `/api/v1` with the session cookie and no token handling. In dev, Vite on `:5173` proxies `/api` to the API on `:5076`.

**Routing and role gating** (`src/App.tsx`): every page sits inside `ConsoleLayout`. `/dashboard`, `/domains`, `/domains/:domainId`, `/threats` and `/alerts` are open to every role; `/clients`, `/notifications` and `/report-sources` need staff; `/users`, `/audit`, `/backup` and `/setup/import` need admin. Gating uses `isStaff`/`isAdmin` from `lib/authz.ts`. This is a UX mirror only: the server enforces the same rules.

**Data access** (`src/lib/api.ts`): pages call `fetchJson<T>(url, init)` directly; there is no query library. It sends `credentials: 'include'`, throws `ApiError(message, status)` using the server's `{ error }` text, and calls the handler registered with `setUnauthorizedHandler` on a 401 from any non-auth path, which drops the user back to login. Response types live in `lib/entities.ts` and `lib/analytics.ts` and mirror the C# DTOs by hand.

| Folder | Contents |
| --- | --- |
| `pages/` | One component per route; tests sit beside them as `<Page>.<aspect>.test.tsx` |
| `components/ui/` | Primitives: `badge`, `button`, `card`, `dialog`, `icon`, `input`, `select`, `table` |
| `components/data/` | `ComplianceBar`, `PolicyBadge`, `SortHeader`, `StatCard`, `TrendChart` |
| `components/` | Shell and feature cards: `ConsoleLayout`, `AuthProvider`, `LoginPage`, `ApiCredentialsCard`, `ConfigImportPanel`, `ReportBackfillCard` |
| `lib/` | API client, types, pure helpers (`domain-grouping`, `format`), hooks (`use-hostnames`, `use-system-status`, `use-page-title`) |

**Design tokens**: CSS variables in `src/index.css` (`--brand` teal-700 `#0c7568`, `--surface-*`, `--text-*`, `--border-*`, `--status-{ok,warn,danger,neutral}-{fg,bg,dot}`) surfaced through the Tailwind theme. Fonts: Space Grotesk (display), Public Sans (body), JetBrains Mono for technical values. The old shadcn light-blue tokens are gone; new UI uses these tokens and the existing primitives.

`src/web/README.md` is still the stock Vite template, although AGENTS.md points to it as the frontend notes.

## Testing and CI

Three suites gate every PR: fast backend unit tests, Docker-backed PostgreSQL integration tests, and the frontend type-check, test and build.

| Suite | Location | Count | Runs on | Covers |
| --- | --- | --- | --- | --- |
| Backend unit | `src/api.tests` | 75 test files | EF Core InMemory, no daemon | Services, parsers (incl. RFC 9990 fixtures), DNS/policy resolution, MTA-STS, digests, config contract, runtime modes |
| Backend integration | `src/api.integration.tests` | 16 test files | Testcontainers PostgreSQL (`PostgresFixture`) | Raw SQL, `ON CONFLICT` dedup, transactions, ingestion tenancy, POP3/S3 sources, push ingest, backfill |
| Frontend | `src/web/src/**/*.test.ts(x)` | 10 files | Vitest + `src/test/setup.ts` | Page behaviour (DomainDetail, Domains grouping), hooks, pure helpers |

Rules that follow from the split: anything touching raw SQL or `MailboxSyncService` transactions must be proven against a real database, because InMemory cannot run either. `ConfigurationContractTests` fails the build when a setting exists in code but not in `docs/ops/configuration.md`. `VersionReferenceTests` fails on version strings older than the chart.

**CI** (`.github/workflows/ci.yml`, on push to `main`, `v*.*.*` tags, PRs, manual):

| Job | Does |
| --- | --- |
| `test` | `dotnet test` both suites; `npm ci`, `tsc --noEmit`, `npm test`, `npm run build` |
| `chart` | `helm lint`, renders every supported value combination, checks guardrails refuse bad ones; on a tag, checks `Chart.yaml` matches and pushes to `oci://ghcr.io/.../charts` plus Artifact Hub metadata |
| `changes` | Decides whether an image build is needed |
| `image` | Multi-arch (amd64 + arm64) Buildx build to GHCR and Docker Hub, OCI labels, stamped commit |

Restore runs in locked mode in CI and the image (`RestoreLockedMode` in `src/Directory.Build.props`), so a stale `packages.lock.json` fails with NU1004; the fix is `dotnet restore --force-evaluate`, never disabling locked mode.

Gap: AGENTS.md requires `npm run lint` to be clean, but CI does not run it.

## Configuration and deployment

Every setting is documented once in `docs/ops/configuration.md`, applies identically on Compose and Kubernetes, and is build-enforced: add a setting there in the same PR that adds it to code.

**Naming**: ASP.NET nested sections, with a double underscore in environment variables (`Worker:ScheduleIntervalSeconds` → `Worker__ScheduleIntervalSeconds`); lists are indexed (`Network__TrustedNetworks__0`). A single underscore is silently ignored.

| Setting | Default | Role |
| --- | --- | --- |
| `Security__CredentialEncryptionKey` | none (required) | AES-256-GCM key for stored mailbox passwords; losing or changing it makes them undecryptable |
| `ConnectionStrings__Default` / `DATABASE_URL` | none (required) | Npgsql string, or a `postgres://` URI that takes priority |
| `APP_MODE` | `api` | `api`, `worker`, `all`, `migrate`, `mta-sts`; anything else fails startup |
| `Database__MigrateOnStartup` | `false` | Web host only; Compose sets it true, Kubernetes uses the migration Job instead |
| `Worker__*` | see doc | Poll interval, batch sizes, timeouts, `EnforceSingleInstance`, `MaxPushedReportRequestBytes` |
| `Retention`, `Backup`, `Email`, `Alerts`, `Digest`, `Branding`, `Dns`, `MtaSts` | see doc | Feature sections |
| `Auth__Oidc__*` | off | `Enabled`, `Authority`, `ClientId`, `ClientSecret`, `Scopes`, `DefaultRole`, `AutoProvision`, ... |
| `Network__*` | off | Forwarded headers; refused when enabled with an empty trust list |
| `OTEL_*` | off | Standard OpenTelemetry variables; SDK registered only when an exporter is set |

**Topologies**

| Target | Files | Shape |
| --- | --- | --- |
| Compose, single container | `deploy/compose.yml` | `APP_MODE=all` + `postgres:18-alpine` |
| Compose, split | `+ deploy/compose.split.yml` | Separate api and worker; worker waits for a healthy app |
| Compose, external DB | `+ deploy/compose.external-db.yml` | Your own Postgres via `DMARC_DB_*` |
| Kubernetes | `deploy/helm/dmarc-analyzer` | App and worker Deployments (worker pinned to one replica), migration Job, optional Postgres StatefulSet, Ingress, `values.schema.json` + validation guardrails |
| App stores | `deploy/appstores/{umbrel,casaos}` | Packaged Compose manifests |
| Development | repo-root `docker-compose.yml` | api + worker + postgres + dev OIDC providers |

**Releases** are tag-driven (`docs/ops/release.md`): `vX.Y.Z` publishes `latest`, `X.Y.Z`, `X.Y` and `sha-<commit>` to GHCR and Docker Hub plus the Helm chart; pushes to `main` publish `edge`; PRs build but never push. The version lives only in `src/Directory.Build.props` and `Chart.yaml`, bumped by `prepare-release.yml`.

## Norms and working agreements

The house rules, from `AGENTS.md`, `CONTRIBUTING.md` and the code itself.

**Git and review**

- `main` is protected: branch, implement, verify, open a PR. Merges are squash merges and branches are not auto-deleted, so stacked PRs must be retargeted by hand (`gh pr edit <n> --base main`) while still open, then rebased on `origin/main`.
- Work in a git worktree, not the shared checkout; diff and stage only the paths you touched, never `git add -A`. A worktree needs the original `.env` copied (a new encryption key breaks stored passwords) and `npm install`.
- PR descriptions list the commands run and their results. Reference issues with `Refs #123`, not `Closes`.
- Feature PRs update `docs/planning/status.md` and `backlog.md`; a new setting updates `docs/ops/configuration.md` in the same PR.
- Large changes start with an issue; the ADRs in `docs/planning/adr/` hold the load-bearing decisions.

**Backend code**

- Modules stay thin; logic goes in `Application/` services that return `ServiceResult<T>`. Every write is audited.
- New endpoints are staff-only unless stamped; viewer access needs `.AllowClientViewer()` **and** service-side scoping through `CanAccessClient`, with cross-tenant ids answering 404.
- Prefer EF LINQ; raw SQL (`DISTINCT ON`, per-group aggregates) stays tenant-scoped and gets integration tests.
- Nullable reference types and XML doc generation are on (`CS1591`/`CS1573` suppressed). Comments explain *why*, not what the line does; every public type carries a `<summary>`.
- Fail loudly on bad configuration rather than defaulting (`APP_MODE`, empty `Network` trust list, missing connection string in `mta-sts`).
- Frozen names: audit event and target types (`mailbox_source.created`, ...) and config-export entity keys are persisted data. Renaming them is a breaking change.
- Exactly one ingestion worker per database; do not assume horizontal scaling of the worker.

**Frontend code**

- TypeScript strict; `npm run build`, `npm test` and `npm run lint` must pass.
- Sentence-case copy, no emoji, plain technical tone with no hype; mono font for domains, IPs and policies.
- Use the design tokens and the existing `ui/` and `data/` primitives.

**Verification**: build, tests and lint for every change; for user-facing changes also run the stack (`docker compose up -d --build`) and check the real app. Ingestion changes (`MailboxSyncService`) must be verified against a real PostgreSQL and the PR must say so.
