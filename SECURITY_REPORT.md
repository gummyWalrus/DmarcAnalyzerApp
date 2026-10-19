# Security and trust review

Commit reviewed: `34e91d9` (v0.14.0, `main`), 548 tracked files.
Reviewer: Claude (static review). Date: 2026-10-19.

## Verdict

**No telemetry, phone-home, update check, analytics, or malicious pattern found.** The only outbound traffic the code can produce is traffic you configure or that DMARC itself requires. Two hardening gaps need your attention before exposing it to the internet (see Findings).

This is a grep-driven review plus targeted reads of the security-relevant files. It is not a line-by-line audit of every file, and it did not run the app or capture network traffic. See "Not verified".

## Outbound network: complete list

Every network client in `src/api` and what it talks to:

| Component | Destination | Triggered by |
|---|---|---|
| `ImapMailboxTransport`, `Pop3MailboxTransport` (MailKit) | Mailboxes you add | You configure a report source |
| `S3ReportSourceTransport`, `S3ObjectStorage` (AWSSDK.S3) | S3 bucket or endpoint you set | You configure an S3 source or `Backup__*` |
| `EmailSender` (MailKit SMTP) | `Email__Host` | You set SMTP (empty by default) |
| `MtaStsPolicyFetcher` (`HttpClient`) | `https://mta-sts.{your-domain}/.well-known/mta-sts.txt` | MTA-STS check on domains you added |
| `DnsTxtResolver`, `DnsMxResolver`, `AuthoritativeDnsClientLocator` (DnsClient) | System resolver, then the authoritative nameservers of your own domains | DNS refresh on domains you added |
| `HostnameResolver` (`Dns.GetHostEntryAsync`) | System resolver (PTR lookups of report source IPs) | Analytics pages |
| OpenTelemetry OTLP exporter | `OTEL_EXPORTER_OTLP_ENDPOINT` | Only if you set it (see below) |
| Npgsql | Your Postgres | Always |
| OIDC handler | `Auth__Oidc__Authority` | Only if you enable OIDC |

Checked and absent: no hardcoded external hosts in backend code (the only URLs are the `mta-sts.{domain}` templates and doc comments). No GeoIP, WHOIS, threat-intel, webhook, Slack or similar integrations. No `api.github.com` call or version-check logic. No instance ID, machine ID or usage counters. No hardcoded public resolvers (8.8.8.8 and similar).

Privacy note, not a defect: PTR and DNS lookups go to your resolver and to your domains' nameservers, so report source IPs and your domain list are visible to whoever runs that resolver, as with any DNS use.

## Telemetry

OpenTelemetry is the only telemetry code (`src/api/Application/Observability/TelemetrySetup.cs`, `TelemetrySettings.cs`).

- **Off by default.** The SDK is never registered unless `OTEL_EXPORTER_OTLP_ENDPOINT` is set or an exporter is set to `console`. `OTEL_SDK_DISABLED=true` forces it off. No endpoint is hardcoded, and `appsettings.json`, both Compose files and the Helm `values.yaml` contain no `OTEL_*` setting.
- It exports to an endpoint only you can name. Nothing reaches the project's authors.
- If you do enable it, logs export with `IncludeFormattedMessage`, `IncludeScopes` and `ParseStateValues`, so log values such as report source IDs and email addresses in audit text leave the process. Point it at a collector you control.
- The frontend (`src/web`) has no analytics, error reporting, beacon, WebSocket, tracking script or third-party CDN. The only `fetch` calls go to same-origin `/api/v1/...`. Fonts are bundled through `@fontsource`, not Google Fonts. The only external URL in the UI is a link to the project's GitHub repository (`use-system-status.ts`), which the browser fetches only if you click it.

## Code-execution and obfuscation checks

Searched `src/` for `Process.Start`, `Assembly.Load`, `Activator.CreateInstance`, `DllImport`, `Emit`, `unsafe`, `eval`, `new Function`, `Type.GetType` and base64 decoding. Hits: base64 for the AES key and password hash handling (`AesGcmCredentialProtector`, `PasswordHasher`), and one test. No dynamic loading, shelling out, native interop or obfuscated blobs. No committed secrets (AWS key, GitHub token, Slack token and private-key patterns: no hits). No seeded users in migrations (`InsertData` absent).

## Supply chain

- **NuGet:** 20 direct packages, all mainstream (Microsoft, AWS, OpenTelemetry, MailKit, Npgsql, DnsClient, Carter, SharpCompress). One niche package: **`DmarcRua` 2.1.1**, a third-party DMARC XML parser; it handles untrusted report XML, so it is worth your own look. Lockfile is committed, and the Docker build and CI restore with `RestoreLockedMode`.
- **npm:** no non-registry `resolved` URLs. The only package with an install script is `fsevents` (optional macOS dependency, standard). Frontend runtime dependencies are React, Radix, Tailwind helpers, `tldts`, `lucide-react`, fonts.
- **Dockerfile:** official `node`, `dotnet/sdk` and `dotnet/aspnet` base images, no `curl | sh`. Runtime image installs `libgssapi-krb5-2` and `curl` (curl is used by the healthcheck). Runs as a non-root user.
- **CI:** all GitHub Actions are pinned to commit SHAs. Secrets used: `GITHUB_TOKEN`, `DOCKERHUB_TOKEN`. One weak spot: the release job downloads `oras` v1.2.3 from GitHub with `curl` and no checksum check (`.github/workflows/ci.yml` ~L259). This affects the maintainers' pipeline, not your runtime.

## Findings (hardening, not malice)

1. **First-run bootstrap race (medium).** `POST /api/v1/auth/register` creates the first account as admin when none exists, with no token. `GET /api/v1/auth/setup` is unauthenticated and tells anyone whether setup is pending. If you expose a fresh instance before completing setup, whoever gets there first owns it. Complete setup on a private network, then expose it.
2. **No login throttling (medium).** The only rate limiter is for pushed report ingest (`ReportIngestRateLimiting.cs`). `/api/v1/auth/login` has no rate limit or lockout, only an audit entry per failure. Put a reverse proxy rate limit in front, or use OIDC with `DisableLocalLogin`.
3. **Default database password (low).** Compose falls back to `postgres` / `postgres` and the Helm chart's database password defaults to `postgres`. Postgres is not published on the shipped compose file, but set `DMARC_DB_PASSWORD` anyway.
4. **Credential encryption is optional (low).** Without `Security__CredentialEncryptionKey`, mailbox passwords are stored in plaintext with a startup warning. The shipped Compose file refuses to start without a key, so this only affects a manual run.
5. **SSRF surface on admin-supplied endpoints (low, admin only).** The MTA-STS fetcher blocks private networks by default (`MtaSts__AllowPrivateNetworks=false`, connect-time check) and does not follow redirects. The S3 endpoint and IMAP/POP3 host fields have no such guard, so whoever can create or edit a source can make the server connect to internal addresses. I did not verify which roles can do that. Expected for a self-hosted tool, but it matters if you give out elevated roles.
6. **`AllowedHosts: "*"`.** Host-header filtering is off. Behind a proxy that is normal; just do not rely on it.
7. **Image tag.** `deploy/compose.yml` uses `ghcr.io/dmarc-analyzer-net/dmarc-analyzer:latest`. Pin a version or digest.

Positive controls seen: session cookie is `HttpOnly`, `SameSite=Lax`, `Secure` on HTTPS; XML parsing sets `DtdProcessing.Prohibit`; archive extraction has entry, total-size and entry-count limits (`ReportPayloadLimits`); mailbox passwords use AES-256-GCM; endpoints are deny-by-default for `client_viewer`; forwarded headers refuse an empty trust list; a security policy with private reporting exists.

## Not verified

- Whether the published image `ghcr.io/dmarc-analyzer-net/dmarc-analyzer` matches this source. Build it yourself with `docker build .` if that matters.
- Transitive dependency contents (58 NuGet and several hundred npm packages). Only direct dependencies and install scripts were checked.
- Runtime behaviour: no traffic capture. To confirm, run it with egress blocked except to your mailbox, DNS and SMTP, and watch for denied connections.
- `src/api/Data/Migrations` and test projects were excluded from the pattern searches.
- Authorization correctness of each endpoint (per-client grant checks) was not reviewed in depth.
