# RFC-20261012-managed-k8s-reference-sites: Production reference deployments of Michelangelo on managed Kubernetes (GKE and OKE)

- **Status:** Draft
- **Author(s):** @sallycr
- **Created:** 2026-10-12
- **Internal ERD:** N/A

---

## Problem statement

Michelangelo ships a Helm chart and a local sandbox (`ma sandbox create`, k3d) that are tuned for evaluation and development. There is no documented, tested, end-to-end configuration for running the control plane (`apiserver`, `controllermgr`, `worker`, `ui` and the Envoy front proxy) with its dependent systems as a production service on a managed Kubernetes platform. Concretely:

- **No production profile.** The base `values.yaml` and `values-k3d.yaml` carry sandbox conveniences (single replicas, in-cluster state, plaintext, shared ServiceAccount). The chart has no documented overlay that turns on production-oriented options.
- **Authentication and authorization are in flight in several places at once.** The default is `DummyAuth` (allow-all), and the documentation runs ahead of the code; RFC #20 describes this in detail and it is not restated here. Since then, michelangelo-ai/michelangelo#2199 has merged pluggable bearer-token hooks, while RFC #16, RFC #20 and an open PR stack (#1852, #1988-#1992) propose overlapping and, in places, incompatible pieces. Nobody owns the question "what is the single supported path, and how is it deployed end to end?"
- **Transport security, workload identity and managed state are not wired through the chart.** TLS options exist in places but are not exercised end to end; the chart uses one ServiceAccount for all workloads; state defaults to in-cluster MySQL and MinIO with static credentials.
- **No cloud-specific guidance, and the clouds differ.** Operators on GKE and on OKE (Oracle Kubernetes Engine) have different managed databases, object-storage APIs, pod identity mechanisms, secret stores and load-balancer behavior. Without a shared structure each would build a divergent, untested configuration.
- **CI exercises only the sandbox configuration.** The integration test runs one k3d cluster on one self-hosted runner with the permissive defaults, so a production-style deployment (authentication on, TLS on, least-privilege workloads) is never tested.

## Motivation

Operators evaluating Michelangelo for real workloads need a reference they can copy, and the project needs living deployments to keep its production story honest. Reference sites on GKE and OKE give maintainers and design partners concrete, continuously exercised targets for the auth, TLS and hardening work already proposed in the community, and give that work a common integration point instead of several independent efforts. Targeting two different managed platforms also keeps the production profile honest about what is cloud-neutral and what is not.

The main risk is regressing the experience that makes Michelangelo easy to try: one command to a working sandbox, no identity provider, no certificates, no cloud account. This RFC therefore treats **sandbox non-regression as a first-class, automatically tested requirement** and makes every production capability an opt-in overlay.

## Goals

- Define a **cloud-neutral production profile** (`values-production.yaml` plus documentation) covering authentication, TLS, least-privilege workloads and externalized state, layered on an unchanged base chart.
- Provide **thin per-cloud overlays** (`values-gke.yaml`, `values-oke.yaml`) that carry only cloud-specific choices, so nothing cloud-specific enters the base chart, the production profile or the sandbox.
- Stand up and continuously operate **reference sites on both GKE and OKE** using those overlays, with a defined minimal first milestone (v0) and an explicit order (see "Scope management").
- Keep `ma sandbox create` a single command with no new prerequisites, no new required values, and unchanged defaults, enforced by an automated CI gate (guardrails SG-1 to SG-6). The guardrails apply equally to both clouds.
- Provide **one authorization path** by composing existing work (#2199, RFC #16, RFC #20 and its PR stack) and proposing how their interfaces are reconciled.
- Add CI coverage that runs the production profile (authentication and TLS enabled) on every relevant change, with a cloud-neutral baseline on kind and periodic runs per cloud.
- Keep deployment-specific identity and policy integrations outside the open-source repository, behind the existing extension seam.

## Non-goals

- Changing any default. The base chart continues to default to `DummyAuth`, no TLS, and the sandbox profile.
- Redesigning the authentication or authorization model. That is the subject of RFC #16 and RFC #20; this RFC only proposes how they compose and what is needed to deploy them.
- Making any single cloud mandatory. Core code and the base chart must not depend on a specific cloud; other managed or self-managed Kubernetes platforms should be able to follow the same structure with their own overlay.
- Specifying a particular identity provider, organization-specific policy, or compliance regime.
- Multi-region, disaster recovery, or autoscaling targets beyond what v0 needs (see open questions).
- Vulnerability reporting or disclosure process; see the repository's `SECURITY.md`.

## High-level architecture

### Design principles

1. **Opt-in, never default.** Production behavior activates only through an overlay or explicit values. Base defaults do not change.
2. **Cloud-neutral core, thin cloud overlays.** The production profile is expressed in terms available on any conformant Kubernetes cluster (Kubernetes RBAC, cert-manager, External Secrets Operator, NetworkPolicy, S3-compatible object-store endpoint). Anything that names a specific cloud service lives in `values-gke.yaml` or `values-oke.yaml` (and the per-cloud sections of the guide). If an overlay needs more than values (for example a code change), that is a signal the seam is wrong and it is raised as an open question, not hidden in the overlay.
3. **One seam, many deployments.** Vendor- or organization-specific logic plugs in through the extension points introduced in #2199 (`auth.Authenticator`, `auth.TokenSource`, optional fx injection). Environment-specific modules live in separate, privately maintained modules that compose the open-source packages; they are not part of this repository.
4. **Options, not decisions.** Managed cloud services are listed as options with their trade-offs. The overlays document the values for each option; operators choose.
5. **Testable claims only.** Each guardrail and each milestone gate is an assertion CI can check. Statements about cloud-service behavior in this RFC are either cited to vendor documentation or marked unverified.

### Component map

```
 Browser / mactl / API clients
           |
   [ Ingress / load balancer + TLS certificate ]       (cloud-specific option)
           |
   [ Envoy front proxy ]  --TLS-->  [ apiserver (2+ replicas) ]
           |                          |        |            |
   [ UI (+ auth proxy, Phase 3) ]     |        |            +--> authorizer (Kubernetes RBAC, SubjectAccessReview)
                                      |        +--> managed MySQL (TLS, private network)
                                      +--> workflow engine (Temporal/Cadence)  <--  [ worker (2+ replicas) ]
                                                                                          |  bearer token (per-project runner ServiceAccount)
   [ controllermgr (leader election) ] --> Kubernetes API, compute clusters, object store (S3-compatible API)
```

| Component | Sandbox (unchanged) | Production profile (cloud-neutral) |
|---|---|---|
| apiserver | 1 replica, `DummyAuth`, plaintext | 2+ replicas, authentication and authorization enabled, TLS listener, probes, own ServiceAccount |
| Envoy | Plaintext, permissive CORS | TLS to the apiserver with certificate verification, scoped CORS including the `authorization` header, no Kubernetes API access |
| UI | No login flow | Behind an authenticating proxy once RFC #16 lands; non-root |
| controllermgr | 1 replica | 2+ replicas with leader election; own ServiceAccount with only the verbs it needs |
| worker | Plaintext, no token | TLS and a bearer token via the #2199 outbound hook; per-project runner identity |
| Relational state | In-cluster MySQL | External MySQL-compatible database with TLS and a dedicated non-root user (cloud option below) |
| Object store | In-cluster MinIO, static keys | S3-compatible endpoint (cloud option below) |
| Pod and network posture | Chart defaults | Pod security `restricted` where workloads allow, `seccompProfile: RuntimeDefault`, default-deny NetworkPolicy with explicit allows (requires a CNI that enforces policy) |
| Supply chain | Floating tags | Digest-pinned images, signed images, SBOMs, vulnerability scanning gating release (staged; see rollout) |

### Per-cloud options

Everything below is an **option, not a decision**. "Verified" means the statement was checked against the cited vendor page while drafting; "unverified" means it is believed but not checked and must be confirmed before the overlay relies on it.

| Concern | GKE option | OKE option | Cloud-neutral fallback |
|---|---|---|---|
| Managed MySQL | Cloud SQL for MySQL. Private connectivity, TLS enforcement, IAM-based database auth: all unverified. | OCI MySQL HeatWave Service, described as fully managed with automated backup and patching ([overview](https://docs.oracle.com/en-us/iaas/mysql-database/doc/overview-mysql-database-service.html), verified). MySQL version compatibility with the chart's schema, high availability, private-subnet access and TLS client settings: unverified. | Self-managed MySQL on the cluster (not recommended for production) |
| Object storage via the S3-compatible API | Cloud Storage XML API with HMAC keys: S3 tools work by pointing at `https://storage.googleapis.com`, with V4 signing ([doc](https://docs.cloud.google.com/storage/docs/interoperability), verified). Static HMAC keys are required on this path. | OCI Object Storage S3 Compatibility API: Customer Secret Keys, path-style and virtual-hosted endpoints, SigV2 not supported ([doc](https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/s3compatibleapi.htm), verified). The page does not list every unsupported S3 operation; whether the chart's client calls and the Ray log collector work against it is unverified. | MinIO or SeaweedFS (see enhancements#25) in-cluster |
| Pod access to cloud APIs without static keys | Workload Identity Federation for GKE: IAM policies for a Kubernetes ServiceAccount, namespace or cluster, short-lived tokens ([doc](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/workload-identity), verified). Note that S3-compatible HMAC access above still uses static keys; keyless access needs a client that speaks the native API (see open questions). | OKE workload identity: IAM policies written against the workload (cluster, namespace, ServiceAccount); available only on enhanced clusters ([doc](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contenggrantingworkloadaccesstoresources.htm), per search summary; the page itself was not fully read, so treat as partially verified). Customer Secret Keys on the S3 path are static keys. | Static credentials from an external secret store, rotated |
| Secret management | Secret Manager via External Secrets Operator (ESO) or the CSI driver. GCP provider support and auth modes: unverified here. | OCI Vault via ESO: ESO documents an Oracle provider supporting user principal, instance principal and workload identity auth ([doc](https://external-secrets.io/latest/provider/oracle-vault/), verified); the provider's stability level was not stated on the page and is unverified. A Vault CSI driver: unverified. | ESO with any supported backend, or Sealed Secrets |
| Ingress, load balancer, certificates | GKE Ingress or Gateway API with Google-managed certificates: unverified. | OCI load balancer provisioned for a Service of type LoadBalancer ([doc](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingloadbalancers-subtopic.htm), existence verified). TLS from a Kubernetes Secret or from the OCI Certificates service; secondary sources describe these as mutually exclusive per Service. Exact annotations: unverified. | cert-manager with an ACME or private issuer, behind any ingress controller; cert-manager is described as an OKE add-on in secondary sources (unverified). Note that a load balancer terminating TLS with a cloud-managed certificate and an in-cluster issuer are different designs; the profile must say which it assumes. |
| Private registry and image pulls | Artifact Registry (unverified). Node pool service accounts handle image pulls under Workload Identity Federation (verified, same doc as above). | OCI Container Registry: `kubectl`-created docker-registry Secret from an auth token, referenced via `imagePullSecrets` in the same namespace as the pods ([doc](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengpullingimagesfromocir.htm), via search summary). Image-signature enforcement is described there; details unverified. | Any registry; the overlay exposes `imagePullSecrets` and registry values only |
| CI test cluster | Short-lived GKE cluster per run or per period. Provisioning time and cost: unverified. | Short-lived OKE cluster per run or per period. Provisioning time and cost: unverified. | **kind** on a CI runner. This is the baseline; per-cloud clusters are periodic, not per-PR. |

### Sandbox non-regression guardrails

Every PR delivered under this RFC must satisfy all of the following, on both clouds' work. SG-5 is enforced automatically.

| # | Guardrail | Testable assertion |
|---|---|---|
| SG-1 | `ma sandbox create` is unchanged | A smoke job runs `ma sandbox create --workflow temporal` on a clean cluster and exits 0 with no new prerequisites, Helm values, images or external services |
| SG-2 | Defaults unchanged | `helm template` with only today's required values renders identical auth-related output: `DummyAuth` wired, no middleware installed, no TLS, unchanged CORS defaults, no identity-provider or auth-proxy pods. No cloud-specific value or annotation appears in default output |
| SG-3 | Production is opt-in via overlay | Production behavior activates only through `values-production.yaml` (plus an optional cloud overlay) or explicit overrides. The base `values.yaml` never sets `apiserver.auth.mode` to anything other than `dummy` (or unset) |
| SG-4 | Chart validations | `templates/validations.yaml` fails fast on inconsistent combinations, for example authentication enabled without an issuer configured, and warns when authentication is enabled without TLS or with a root-user sandbox security context |
| SG-5 | Automated sandbox gate | A CI job runs on every PR touching the auth packages, the chart, the sandbox CLI, or the code-generation templates. It creates the sandbox on an ephemeral cluster, performs create/get/list/delete on a Project and a Pipeline, confirms a pipeline run reaches `SUCCEEDED`, confirms the UI responds, and confirms an unauthenticated API call still succeeds under default settings |
| SG-6 | Optional sandbox secure mode | An optional `--auth` flag (or `values-sandbox-secure.yaml` overlay) deploys a bundled identity provider and auth proxy so contributors can exercise the full flow locally. Additive only, never the default path |

### Composition with RFC #16, RFC #20, and #2199

The existing work covers different layers; this RFC proposes one path through them. The auth path is identical on both clouds; only the identity provider and token plumbing values differ per environment.

| Input | Status (verified 2026-10-12) | Layer |
|---|---|---|
| michelangelo#2199 | Merged | Transport: bearer-token YARPC inbound/outbound middleware, `Authenticator` and `TokenSource` interfaces, `Principal` in request context, optional fx injection. No behavior change unless middleware is provided |
| RFC #16 (Human Identity) | Open | How humans obtain tokens: oauth2-proxy with PKCE for the UI, device grant for `mactl`, bundled identity provider for sandbox |
| RFC #20 (OIDC and Kubernetes RBAC) | Open | What the apiserver does with an identity: authorization through `SubjectAccessReview`, Get/List enforcement, per-project runner ServiceAccounts |
| michelangelo#1852, #1988-#1992 | Open stack implementing RFC #20, each stacked on the previous | Handler enforcement, upstream authenticator/authorizer contracts, OIDC authenticator, `k8s-rbac` mode, worker runner tokens, chart and guide. Default stays `dummy` throughout. Several PRs in the stack currently conflict with `main` |

Proposal, subject to maintainer decision:

- **Transport layer: keep #2199's middleware hooks.** They are merged, vendor-neutral and additive. The reference OIDC authenticator and the reference worker `TokenSource` plug into them, so deployment-specific modules can keep composing the same seam. Which authenticator contract sits behind the middleware is the reconciliation decision below.
- **Handler layer: RFC #20's authorization model.** Generated handlers read the `Principal` from context and obtain an allow/deny from a Kubernetes RBAC authorizer, which is the single authorization code path. `DummyAuth` behavior is preserved by an allow-all authorizer as the default.
- **Role convenience from RFC #16 layers on top.** RFC #16's `admin`/`user` roles become an optional provisioning step (a group-to-role mapping that creates RoleBindings to the shipped viewer/editor ClusterRoles), not a second enforcement mechanism. This is the layering RFC #20 already proposes; RFC #16's author should confirm.
- **Worker identity.** The reference `TokenSource` mints short-lived per-project runner tokens (RFC #20's design), and #2199's outbound middleware attaches them. This relies only on Kubernetes APIs, so it is cloud-neutral.
- **Interface reconciliation is an explicit decision** (see open questions): either adapt the open stack onto #2199's `Authenticator`/`TokenSource` contracts, or retire those in favor of the `k8s.io/apiserver` contracts used by #1988. #1988 as written is a breaking change for out-of-tree implementations of the previous `auth.Auth` interface, including anything built on #2199's hooks, and the stack and #2199 both install a middleware on the worker's API client. Reading the two diffs, inbound token handling would occur twice when both are present (the middleware validates the token before the handler, and the stack's handlers resolve identity from the bearer token again); this should be confirmed with a spike. Preliminary recommendation, for maintainers to confirm: adopt the `k8s.io/apiserver` contracts as the single authenticator contract and make #2199's middleware a thin adapter over it, which avoids authenticating twice and keeps one place for deployment-specific modules to plug in.
- **Ordering constraint** from RFC #20 is load-bearing: enforcement must not be documented as enabled for any deployment until the UI and `mactl` can obtain tokens. Hooks, validator, handler enforcement and authorizer can land behind the allow-all default at any time; UI login and `mactl login` precede the production flip.

### Scope management: which cloud first

**Proposal: one cloud first, the other next, with the same acceptance gate.**

1. Build `values-production.yaml` and prove it cloud-neutrally on kind in CI (the auth integration lane below). This is on the critical path for both clouds and carries most of the work.
2. **Reference site v0 on GKE first.** Reasoning: the repository already contains a native GCS blobstore package under `go/base/blobstore/gcs` and the chart's object-storage comments list `storage.googleapis.com` among example endpoints, so fewer unknowns are expected on the object-store path (whether the chart can select the native client is itself an open question). One cloud first keeps the first overlay small and lets the cloud-neutral boundary be tested by the second overlay rather than guessed.
3. **Reference site v0 on OKE next (v0.1)**, using the identical acceptance gate. Its main unknowns are listed in the options table: S3-compatibility of the client paths, the managed MySQL's TLS and version behavior, and certificate handling at the load balancer. Starting the OKE spikes in parallel with GKE v0 is encouraged because they are inexpensive.

Doing both at once is rejected for v0 because two cloud-specific unknowns multiply each other while the shared profile is still unproven. If the environment owners' availability inverts the order, nothing in the architecture depends on it. The final order is an open question for the owners.

### Reference site v0 (per cloud)

The smallest authenticated, TLS-protected instance:

- Authenticated API: tokens validated, unauthenticated requests rejected.
- TLS at ingress and to the apiserver, with certificate verification at the proxy.
- External managed relational state with TLS and a non-root database user.
- Object storage reached through the S3-compatible endpoint, with credentials sourced from the secret store (keyless access only if the native-client question is resolved).
- Per-component ServiceAccounts.
- Viewer and editor roles enforced through Kubernetes RBAC.
- Worker pipeline runs authenticate to the apiserver with per-project runner tokens.

v0 does not require the browser login flow (operators can use port-forwarding with a manually obtained token), `mactl login`, the bundled sandbox identity provider, NetworkPolicy, autoscaling, audit-log output, or supply-chain hardening. Those follow in later phases.

v0 acceptance gate, identical for both clouds:

1. An unauthenticated API call returns `Unauthenticated`.
2. A valid token with the correct RoleBinding can create, get, list and delete resources in its namespace.
3. A RoleBinding in a different namespace yields `PermissionDenied`.
4. A pipeline run completes using a runner token.
5. The apiserver serves TLS and the proxy verifies it.
6. No credentials are inlined in the overlay; database and object-store credentials come from the secret store.

### CI plan

| Lane | Trigger | What it runs |
|---|---|---|
| Sandbox smoke (SG-5) | Every PR touching the paths in SG-5 | Ephemeral kind cluster, base chart, `ma sandbox create`, CRUD, a pipeline run, UI reachable, unauthenticated call succeeds |
| Auth integration (cloud-neutral) | Nightly, and on PRs touching auth code or auth chart templates | Ephemeral kind cluster with `values-production.yaml`, cert-manager self-signed CA, an OIDC issuer for tests. Asserts: unauthenticated request rejected; valid token plus RoleBinding succeeds; wrong-audience token rejected; cross-namespace access denied; worker run with runner token succeeds; plaintext port not served |
| Cloud lane: GKE | Periodic (weekly or pre-release) | Short-lived cluster with the GKE overlay: managed database, object store, certificates, replicas of at least 2, leader election; node drain with PodDisruptionBudgets, leader failover, database connection behavior under load; torn down after each run |
| Cloud lane: OKE | Same cadence, after GKE v0 | Same tests with the OKE overlay. Quotas and provisioning time unverified |
| Existing k3d lane | Unchanged | Kept as an optional heavy-pipeline lane; not the security or functional gate |

Cost, quotas and runner sizing for the cloud lanes have not been measured and are open questions.

## APIs and CRDs

No CRD or proto changes are introduced by this RFC. The public surface it adds or touches:

- **Helm values.** Overlays `helm/michelangelo/values-production.yaml`, `values-gke.yaml` and `values-oke.yaml`; values introduced by RFC #20's stack (`apiserver.auth.mode`, `apiserver.auth.oidc.*`, per-project runner values); TLS values for the apiserver, worker, database and workflow-engine clients; registry and `imagePullSecrets` values. Names are illustrative and will follow whichever of RFC #16 and RFC #20 naming prevails.
- **Chart templates.** Per-component ServiceAccounts, viewer/editor ClusterRoles (from the RFC #20 stack), probes, PodDisruptionBudgets, NetworkPolicies, optional ServiceAccount and Service annotations passed through from values, all gated by values. No template references a cloud by name.
- **Chart validations.** New checks in `templates/validations.yaml` (SG-4).
- **Sandbox CLI.** Optional `--auth` flag on `ma sandbox create` (SG-6). No change when the flag is absent.
- **Go extension points.** Uses #2199's `Authenticator` and `TokenSource`; adds reference implementations. Whether the authorizer seam is a custom interface or the `k8s.io/apiserver` contract is an open question.
- **Documentation.** One deployment guide with a cloud-neutral part and per-cloud sections; corrections to the operator guides so that documented controls match shipped behavior.

## Alternatives considered

### Alternative A: Authenticate only at the gateway (Envoy `jwt_authn` / `ext_authz`)

**Pros:** No application code changes; works for any backend behind the proxy; quick to enable.
**Cons:** Authentication happens only at one hop, so in-cluster callers (the worker, any direct gRPC client) and handler-level authorization are unaffected; the apiserver still has no notion of the caller, so per-project authorization and audit attribution are not possible; the UI-to-API path needs separate treatment.
**Why not chosen:** It does not provide identity inside the apiserver, which RFC #20 depends on. The gateway may still validate tokens as defense in depth, and the production profile does not preclude it.

### Alternative B: Mutual TLS or a service mesh for identity

**Pros:** Strong workload identity between services; transparent encryption in transit; established tooling.
**Cons:** Addresses service-to-service identity but not human identity or per-project authorization; adds a mesh dependency that sandbox users must not be forced to adopt; behavior differs across platforms and not every deployment has a mesh.
**Why not chosen:** Complementary rather than a replacement. A mesh can be used by operators who have one; the reference profile does not require it. Revisit when service-to-service requirements are better specified.

### Alternative C: Keep deployment sandbox-only and document hardening steps

**Pros:** No new code, CI or maintenance; no risk to sandbox.
**Cons:** Operators must assemble and test their own production configuration on each cloud; the project cannot know whether the auth work in flight composes into a working deployment; the CI gap remains.
**Why not chosen:** It leaves the central problem unsolved. The overlay approach gives the benefits with the sandbox guardrails limiting the downside.

### Alternative D: Fork or maintain a private chart for production

**Pros:** Complete freedom to diverge; no risk to the public chart.
**Cons:** The fork drifts from upstream; fixes made for a reference site do not reach the community; the public chart's production story never improves.
**Why not chosen:** The goal is a shared, public reference. Only environment-specific values and modules should live outside the repository, composed over the public chart and packages.

### Alternative E: Change defaults to secure-by-default

**Pros:** Every install gets the stronger posture without opting in.
**Cons:** Breaks the single-command sandbox experience, requires an identity provider and certificates for evaluation, and is a breaking change for existing users.
**Why not chosen:** Conflicts with the non-regression requirement. RFC #20 also keeps `DummyAuth` as the default.

### Alternative F: One chart or one profile per cloud

**Pros:** Each can use its cloud's native features freely; simpler per-cloud documentation.
**Cons:** Duplicates the auth, TLS and hardening configuration, which are cloud-neutral and would drift; makes cloud-specific values leak toward the base chart; makes a third platform costly to add.
**Why not chosen:** A cloud-neutral profile plus thin overlays keeps the shared configuration in one place and keeps cloud specifics contained.

### Alternative G: Target a single cloud only

**Pros:** Smallest scope; fastest to a working reference.
**Cons:** Cloud-neutrality of the profile would be asserted rather than tested; the community on the other platform gets no reference.
**Why not chosen:** The proposed sequencing (GKE first, OKE next) captures most of the scope benefit while still testing neutrality.

## Open questions

- [ ] **Which cloud first.** This RFC proposes GKE v0, then OKE v0.1, with the same gate. Owners to confirm or invert.
- [ ] **Environment ownership.** Who owns, operates and funds each reference site; whether each is publicly reachable; budget and quotas for the cluster, database and object store per cloud.
- [ ] **Identity provider per environment** (issuer, audience, claim and group mapping). Environment-specific details stay in privately maintained modules.
- [ ] **Interface reconciliation.** Adapt the open PR stack onto #2199's `Authenticator`/`TokenSource` contracts, or retire those in favor of the `k8s.io/apiserver` contracts? Who rebases the stack, and in what order (#1852 first)?
- [ ] **Double authentication.** When #2199's inbound middleware and the stack's handler-level token resolution are both active, is the token validated twice, and does that matter? Needs a spike.
- [ ] **Authorization model.** Confirm the layering of RFC #16's roles over RFC #20's RBAC (one enforcement path), and converge value naming (`auth.rbac.groupRoleMapping` vs `apiserver.auth.groupRoleMapping`).
- [ ] **Authorization seam shape.** Custom authorizer interface versus the `k8s.io/apiserver` `authorizer.Authorizer` contract.
- [ ] **List semantics** for namespace-scoped users (deny cluster-wide List, or fan out per-namespace checks); inherited from RFC #20.
- [ ] **UI token path.** Does the auth proxy forward the original token, or does the apiserver trust proxy-supplied headers? (See RFC #20.)
- [ ] **Object storage on each cloud.** (1) Can the chart select the native GCS client, and is keyless access via Workload Identity then possible? Without it, GKE uses static HMAC keys through the S3 path. (2) Do all object-store client calls in the apiserver, worker and the Ray log collector work against OCI Object Storage's S3 Compatibility API, including addressing style and region settings? (3) Can static keys be rotated safely from the secret store?
- [ ] **Managed MySQL on each cloud.** Version compatibility with the chart's schema and migrations, TLS client configuration, private connectivity, and high availability behavior for both Cloud SQL and OCI's MySQL service.
- [ ] **Certificates on each cloud.** Terminate TLS at the load balancer with a cloud-managed certificate, or in-cluster with cert-manager? The overlays must state which, and the apiserver-side TLS requirement differs.
- [ ] **Secret store integration.** Confirm ESO provider maturity and supported auth modes for each cloud's secret service; decide ESO versus a CSI driver as the documented default.
- [ ] **Registry and image pulls** on each cloud, and whether image-signature enforcement is part of v0.
- [ ] **Spikes before dependent PRs.** (1) Whether the HTTP `Authorization` header passes through the Envoy JSON transcoder to the YARPC `authorization` header; (2) apiserver TLS startup behavior with the chart's existing TLS option; (3) UI container running as non-root.
- [ ] **Per-project runner ServiceAccount provisioning:** Helm over a known project list, or a project-creation hook.
- [ ] **Cluster shape per environment** (private cluster, node pools, policy-enforcing CNI, pod security level), managed versus in-cluster workflow engine, compliance requirements, and load targets (replicas, database sizing, workflow throughput).
- [ ] **Cloud CI lanes:** hosting accounts, budget, expected per-run cost and provisioning time (none measured), and whether they run from public CI.
- [ ] **Sandbox-impact exceptions.** Several hardening items (NetworkPolicy, database TLS, workflow-engine TLS, seccomp, ServiceAccount split) touch resources the sandbox also uses. Are overlay-only variants acceptable, or must the sandbox overlay re-set values explicitly?
- [ ] **Out-of-tree consumers.** Are there downstream modules built on #2199's hooks that #1988's breaking change would affect?

## Rollout strategy

### Phasing and PR split

The "Sandbox impact" column records what each change does to the default sandbox experience; "Needs care" means the sandbox overlay may need explicit values, and SG-5 must pass.

| Phase | Change | Cloud scope | Sandbox impact |
|---|---|---|---|
| 0 | Fix the sandbox nightly (database pod guard) so the CI baseline is green | Neutral | None |
| 0 | Correct operator docs (authentication, compliance) to match shipped behavior; update supported versions in `SECURITY.md` | Neutral | None |
| 0 | Add `authorization` to the Envoy CORS allowed headers; make the CORS origin an explicit setting | Neutral | Additive; sandbox values set origins explicitly |
| 0 | Add the sandbox smoke gate (SG-5) and chart validations (SG-4) | Neutral | None; the safety net for later phases |
| 1 | Handler enforcement on Get/List (#1852); upstream contracts (#1988); OIDC authenticator (#1989); `k8s-rbac` mode (#1990); runner tokens (#1991); chart and guide (#1992), after interface reconciliation | Neutral | None under the default; allow-all behavior pinned by test |
| 1 | Reference authenticator and `TokenSource` on #2199's hooks, with token caching, if not already covered by the stack | Neutral | None; not wired by default |
| 2 | Apiserver TLS listener; TLS verification at Envoy; worker outbound TLS plus token | Neutral | None (TLS off by default); Envoy verification needs care |
| 2 | Per-component ServiceAccounts | Neutral | Needs care; sandbox overlay may re-set the shared account |
| 2 | Probes and PodDisruptionBudgets, with the health endpoint exempt from authentication | Neutral | Additive |
| 2 | `values-production.yaml`, auth integration lane on kind | Neutral | None (overlay only) |
| 2 | `values-gke.yaml`, GKE deployment guide, **GKE v0 gate** | GKE | None (overlay only) |
| 2b | `values-oke.yaml`, OKE deployment guide, **OKE v0.1 gate** | OKE | None (overlay only) |
| 3 | UI login through auth proxy and PKCE; whoami endpoint; `mactl login`; bundled sandbox identity provider behind `--auth` (RFC #16); group-to-role provisioning | Neutral | Additive; opt-in |
| 3 | Audit-log implementation reading the request `Principal` | Neutral | Additive |
| 4 | NetworkPolicy; database and workflow-engine client TLS; seccomp profile | Neutral | Needs care |
| 4 | Digest-pinned images, image signing and SBOMs, broader vulnerability scanning, `values.schema.json` | Neutral | None / additive |

Phases 0 and 1 are independent of any cloud. Phase durations are not estimated here; they depend on maintainer capacity and the open questions above.

### Feature flags

No new global flags. Production behavior is selected by the overlays and by `apiserver.auth.mode`; the sandbox secure mode by `--auth`.

### Migration path

Existing users change nothing. Operators who want production behavior adopt `values-production.yaml` plus their cloud overlay and follow the guide, enabling enforcement only after clients can obtain tokens (ordering constraint above).

### Rollback

- Authentication and authorization: set `apiserver.auth.mode: dummy` (or remove the key). Authorization decisions are computed at request time from live RoleBindings, so no data migration is involved.
- TLS: disable the TLS value to return to plaintext.
- ServiceAccount split: revert to the shared account; any component missing a needed permission fails immediately, so test before merging.
- Overlays: stop using them; the base chart returns to sandbox-safe defaults.
- Workflow-engine authorization, if enabled: disable it on the engine and restart if workers are rejected.

### Risks

- Enabling enforcement before clients can obtain tokens breaks the UI or CLI. Mitigated by the ordering constraint, the default, and documentation.
- A cloud-specific behavior discovered late (for example an object-store API incompatibility) forces a code change rather than a values change. Mitigated by the sequencing, early spikes, and the principle that such findings go back to the open questions rather than into hidden overlay workarounds.
- Authorization-check latency on every RPC. Mitigated by the decision cache proposed in RFC #20; to be measured in the cloud lanes.
- Per-project runner ServiceAccount growth. Mitigated by automated provisioning and cleanup (open question).
- The open stack's conflicts with `main` and with #2199 delay Phase 1. Mitigated by deciding the reconciliation first.

## References

- michelangelo-ai/michelangelo#2199: pluggable bearer-token auth for apiserver and worker (merged)
- michelangelo-ai/michelangelo#1852, #1988, #1989, #1990, #1991, #1992: open implementation stack for RFC #20
- michelangelo-ai/enhancements#16: Human Identity
- michelangelo-ai/enhancements#20: OIDC token validation and Kubernetes-RBAC authorization behind the Auth interface
- michelangelo-ai/enhancements#25: Migrate object storage from MinIO to SeaweedFS
- Repository `SECURITY.md`: how to report vulnerabilities
- Vendor documentation cited in the per-cloud options table: [GCS interoperability](https://docs.cloud.google.com/storage/docs/interoperability), [Workload Identity Federation for GKE](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/workload-identity), [OCI Object Storage S3 Compatibility API](https://docs.oracle.com/en-us/iaas/Content/Object/Tasks/s3compatibleapi.htm), [OKE workload access to OCI resources](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contenggrantingworkloadaccesstoresources.htm), [External Secrets Operator: Oracle Vault](https://external-secrets.io/latest/provider/oracle-vault/), [OCI MySQL HeatWave overview](https://docs.oracle.com/en-us/iaas/mysql-database/doc/overview-mysql-database-service.html), [OKE load balancers](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingloadbalancers-subtopic.htm), [OKE image pulls from OCIR](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengpullingimagesfromocir.htm)
