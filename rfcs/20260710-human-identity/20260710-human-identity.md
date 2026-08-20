# RFC-20260710-human-identity: Human Identity

- **Status:** Draft
- **Author(s):** @craigmarker
- **Created:** 2026-07-10
- **Internal ERD:** <!-- none yet -->

---

## Problem statement

The Michelangelo platform currently has no real authentication for human users. Concretely:

- The Go apiserver's `Auth` interface is implemented by `DummyAuth`, which always returns `true` and is not even invoked on the request path.
- The CLI (`mactl`) connects to the apiserver over plain gRPC with no credentials attached.
- The web UI's `UserProvider` context carries only a `timeZone` — there is no notion of a logged-in user, no avatar, no name.
- There is no whoami endpoint (`/api/v1/me` or equivalent).
- No oauth2-proxy or equivalent is deployed in front of any surface.
- No auth-related annotations exist on the ingress.

This means any human who can reach the API server or the web UI has full, unaudited access to every project and resource on the platform, with no way to know who did what.

## Motivation

As Michelangelo moves from a single-tenant or trusted-network deployment model toward being installed by multiple teams and organizations (including external operators), the lack of authentication becomes a blocker:

- Operators cannot safely expose the platform beyond a fully trusted network.
- There is no way to attribute actions (who deployed this pipeline, who deleted this model) for debugging or compliance.
- Every downstream feature that depends on "who is the current user" (personalization, scoped resource lists, approvals, audit trails) is blocked until an identity primitive exists.

This RFC scopes strictly to **human identity** — Web UI and CLI. Service-to-service authentication (Kubernetes service account tokens, mTLS between internal services) is a related but separate concern with different threat models and rollout mechanics, and is explicitly out of scope here to keep this RFC reviewable.

## Goals

- Establish OIDC as the single identity backbone for all human-facing surfaces.
- Authenticate the Web UI via the Authorization Code + PKCE grant (RFC 7636), fronted by oauth2-proxy at the ingress layer.
- Authenticate the CLI (`mactl`) via the Device Authorization Grant (RFC 8628), with credentials cached locally and attached to gRPC calls.
- Replace `DummyAuth` in the Go apiserver with a real token validator that checks signatures against the IdP's JWKS endpoint and extracts `sub`/`email`/`groups` claims.
- Ship a minimal but real RBAC model (`admin` / `user` roles, derived from OIDC `groups`) that plugs into the existing `Auth` interface.
- Replace `DummyAuditLog` with a real implementation that records identity, action, resource, and result for every authenticated request.
- Support three deployment tiers (local dev, sandbox, production) so auth can be exercised end-to-end before it reaches operators' clusters.

## Non-goals

- Service-to-service authentication (Kubernetes SA tokens, mTLS) — tracked as a separate RFC.
- Fine-grained permissions beyond the two built-in roles (e.g., per-project custom roles) — future enhancement once the RBAC foundation lands.
- Multi-tenancy / organization-level isolation — separate RFC.
- A token management UI (API key generation, rotation, personal access tokens) — separate feature.

## High-level architecture

One OIDC-compliant identity provider (Keycloak, Dex, or any operator-supplied IdP), with a different OAuth2 grant type per surface:

| Surface | Grant Type | RFC | Flow |
|---|---|---|---|
| Web UI | Authorization Code + PKCE | RFC 7636 | oauth2-proxy at the k8s ingress handles the OIDC login, sets a session cookie, and a BFF session endpoint (`/api/v1/me`) returns the caller's identity |
| CLI (`mactl`) | Device Authorization | RFC 8628 | `mactl login` prints a verification URL + code; once the user approves in a browser, the CLI receives a token and caches it in `~/.ma/credentials`, attaching it as gRPC metadata on subsequent calls |

This mirrors the pattern used by Kubernetes (kubectl + Dashboard), GitLab (web + CLI + API), and Backstage (web + backend plugins): one IdP, multiple grant types tailored to each client's ability to handle redirects and browsers.

### Key components

**1. IdP integration.** The platform deploys with a configurable OIDC issuer. The Helm chart includes an optional Dex/Keycloak subchart for standalone deployments; enterprise operators can instead point to their org's existing IdP (Okta, Azure AD, etc.) via Helm values.

**2. Web UI authentication.**
- oauth2-proxy runs at the ingress layer (sidecar or standalone pod) and handles the OIDC Authorization Code + PKCE flow.
- On success, it injects `X-Auth-Request-User`, `X-Auth-Request-Email`, and `X-Auth-Request-Groups` headers onto proxied requests.
- A BFF endpoint, `/api/v1/me`, reads those headers and returns OIDC-standard claims: `{ name, email, picture, groups }`.
- The frontend's `CurrentUserProvider` fetches `/api/v1/me` on load and populates the existing `UserProvider` context, which the `NavigationBar` uses to render an avatar/name.
- Graceful degradation: when auth is not configured (local dev), the endpoint returns 401 and the nav bar simply omits the user section — no hard dependency on auth being present.

**3. CLI authentication.**
- `mactl login` initiates the Device Authorization Grant, showing a verification URL and user code, then polls the token endpoint until the user approves in a browser.
- The access token and refresh token are stored in `~/.ma/credentials` with file permissions `0600`.
- Every subsequent `mactl` command attaches the access token as `authorization` gRPC metadata.
- `mactl logout` clears stored credentials.
- Token refresh happens transparently on expiry, using the cached refresh token.

**4. Go apiserver enforcement.**
- `DummyAuth` is replaced with a real OIDC token validator.
- A gRPC unary interceptor extracts the bearer token from request metadata (CLI path) or reads the `X-Auth-Request-*` headers forwarded by oauth2-proxy (web path).
- The validator checks the token's signature against the IdP's JWKS endpoint and extracts `sub`, `email`, and `groups` claims.
- Resulting identity is attached to the request context for use by downstream handlers, RBAC checks, and audit logging.

**5. RBAC.**
- Two built-in roles to start: `admin` (full platform access — manage projects, configure platform settings, view all resources) and `user` (project-scoped access — create/read/update resources within assigned projects).
- Roles are derived from the OIDC `groups` claim (e.g., group `michelangelo-admins` maps to the `admin` role); the group-to-role mapping is configurable via Helm values.
- The existing `Auth` interface — `UserAuthorized(ctx, namespace, action, resource)` — already has the right shape; this RFC proposes a real implementation that checks the caller's role against the requested action instead of always returning `true`.
- Fine-grained, per-project or custom roles can build on this foundation later without changing the interface.

**6. Audit logging.**
- `DummyAuditLog` is replaced with a real implementation.
- Every authenticated request logs timestamp, user identity (`sub`/`email`), action, resource, and result.
- Identity is propagated from the auth context through the request lifecycle so audit entries are always attributable.

### Deployment tiers

| Tier | IdP | oauth2-proxy | Auth enforcement |
|---|---|---|---|
| Local dev (`yarn dev`) | None | None | Disabled — `DummyAuth` remains, no user section in nav bar |
| Sandbox (k3d) | Bundled Dex | Deployed via Helm | Full — exercises the real auth flow locally |
| Production (k8s) | Operator's IdP (Okta, Azure AD, etc.) | Deployed via Helm | Full |

## APIs and CRDs

**New REST endpoint:**

`GET /api/v1/me` — returns `{ name, email, picture, groups }` from proxy-injected headers, or `401` when unauthenticated or when auth is disabled.

**gRPC interceptor (apiserver-internal, no new proto surface):**

Validates bearer tokens or forwarded auth headers and populates request context with user identity.

**New CLI subcommands:**

- `mactl login` — initiates Device Authorization Grant flow and caches credentials locally.
- `mactl logout` — clears stored credentials.

**Helm values additions (illustrative, not final):**

```yaml
auth:
  enabled: false
  oidc:
    issuerUrl: ""
    clientId: ""
  oauth2Proxy:
    enabled: false
  rbac:
    groupRoleMapping:
      michelangelo-admins: admin
```

## Alternatives considered

### Alternative A: Static API tokens (no OIDC)

**Pros:** Simple to implement; no dependency on an external IdP; works offline.

**Cons:** No standard login UX for the web UI; token issuance/rotation becomes a bespoke system we'd have to build and secure ourselves; doesn't integrate with organizations' existing SSO, which most enterprise operators require.

**Why not chosen:** Punts the hardest problems (session UX, token lifecycle, SSO integration) onto us instead of leveraging a well-understood standard.

### Alternative B: Different identity mechanisms per surface (e.g., session cookies for web, mTLS client certs for CLI)

**Pros:** Each mechanism could be tuned to its surface.

**Cons:** Doubles the implementation and operational surface area (two credential lifecycles, two revocation paths, two things to document); doesn't match how the CLI is actually used (interactive, human-driven) which maps naturally onto Device Authorization Grant.

**Why not chosen:** OIDC already defines a grant type suited to each surface (Authorization Code + PKCE for browsers, Device Authorization for input-constrained/non-browser clients), so a single IdP with two grant types gives us one identity model with less bespoke code.

### Alternative C: Build our own IdP

**Pros:** Full control, no external dependency.

**Cons:** Significant, ongoing security-sensitive engineering effort (credential storage, MFA, password reset, session management) that is not Michelangelo's core value proposition, and most enterprise operators already have an IdP they'd prefer to reuse.

**Why not chosen:** OIDC compliance lets us plug into whatever IdP an operator already runs (or ship a bundled Dex/Keycloak for standalone use) instead of owning identity infrastructure ourselves.

## Open questions

- [ ] Which OIDC claim(s) should be treated as the canonical stable user identifier for audit logs — `sub` alone, or `sub` + issuer, to handle multi-IdP futures?
- [ ] Should the bundled Dex/Keycloak subchart be the default for new installs, or opt-in only, given the added operational footprint?
- [ ] Token lifetime and refresh policy for `mactl` — how long can a cached CLI credential remain valid before requiring re-login?
- [ ] Where does oauth2-proxy terminate relative to the ingress — sidecar per service, or a single shared instance in front of all HTTP-facing services?

## Rollout strategy

**Phase 1 (alpha):** Land the OIDC token validator and RBAC scaffolding behind `auth.enabled: false` (default off). `DummyAuth` and `DummyAuditLog` remain the default; no behavior change for existing installs.

**Phase 2 (beta):** Wire up the bundled Dex option and oauth2-proxy in the sandbox (k3d) Helm profile so the full flow can be exercised end-to-end in CI and sandbox before any production rollout. The web UI's `CurrentUserProvider` and `mactl login`/`logout` ship in this phase, guarded by graceful degradation when `/api/v1/me` returns 401.

**Phase 3 (GA):** Document the Helm values needed to point at an operator's own IdP; publish the group-to-role mapping configuration; recommend (but do not force) `auth.enabled: true` for production deployments.

**Migration path:** `DummyAuth` remains available indefinitely as an explicit opt-out for development and trusted-network deployments. Auth is opt-in via `auth.enabled: true` in Helm values. When disabled, all endpoints remain open (today's behavior, unchanged). When enabled, unauthenticated requests receive `401`.

**Rollback:** Setting `auth.enabled: false` and re-deploying immediately reverts to today's open-access behavior; no data migration is involved since RBAC state is derived from OIDC group claims at request time rather than stored.

## References

- [RFC 7636 — Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 8628 — OAuth 2.0 Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628)
- [oauth2-proxy project](https://oauth2-proxy.github.io/oauth2-proxy/)
- Kubernetes authentication model (kubectl + Dashboard) as prior art for one-IdP/multiple-grant-types
- GitLab and Backstage authentication architectures as prior art
