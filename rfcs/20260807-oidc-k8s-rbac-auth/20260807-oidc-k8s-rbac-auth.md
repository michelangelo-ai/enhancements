# RFC-20260807-oidc-k8s-rbac-auth: OIDC token validation and Kubernetes-RBAC authorization on the k8s.io/apiserver authenticator and authorizer contracts

- **Status:** Draft
- **Author(s):** @apurvapatkeshwar
- **Created:** 2026-08-07
- **Revised:** 2026-08-31 -- adopted the k8s.io/apiserver `authenticator`/`authorizer` contracts in place of the bespoke `Auth` interface, and rescoped worker machine identity to per-project runner ServiceAccounts
- **Internal ERD:** N/A (external contribution)

---

## Problem statement

The control plane's authentication and authorization surface is a single 55-line file, `go/auth/auth.go`. It defines the `Auth` interface (`UserAuthorized(ctx, namespace, Action, resourceType) (bool, error)` and `UserAuthenticated(ctx) (bool, error)`, declared in that order) and ships exactly one implementation, `DummyAuth`, whose two methods each return `true, nil` unconditionally. `go/cmd/apiserver/main.go:38` wires `auth.DummyAuthModule` as the only auth module in the fx graph; there is no config flag, build tag, or Helm value that selects anything else.

The mutating RPC handlers call both methods before doing any work. For example, `proto-go/api/v2/cluster.pb.kubeyarpc.go` calls `c.auth.UserAuthenticated(ctx)` at line 195 and `c.auth.UserAuthorized(ctx, projectName, authapi.Create, "Cluster")` at line 204 inside `CreateCluster`, and the same pattern repeats for Update/Delete/DeleteCollection across every resource type the API server exposes: Cluster, Model, ModelFamily, Pipeline, PipelineRun, Project, RayCluster, RayJob, SparkJob, Deployment, InferenceServer, Revision, CachedOutput, EvaluationReport, TriggerRun.

The read handlers call neither. `GetCluster` and `ListCluster` -- and the Get/List handlers of every other resource type -- contain zero auth calls, and the omission is baked into the code generator: in `go/kubeproto/templates/crd_svc.tmpl` (the handler template `templates.go` embeds), the Create/Update/Delete/DeleteCollection sections each emit the authentication-and-authorization block, and the Get/List sections emit none. The data to enforce with is already there -- both `GetClusterRequest` and `ListClusterRequest` carry `Namespace`, and the handlers read it today for logging tags -- it is simply never passed to an auth call. With `DummyAuth` wired the difference is invisible, since everything is allow-all anyway; but it means a real `Auth` implementation dropped behind the interface, with no other change, would enforce writes while leaving every read in the system wide open. Closing that generator gap is in scope for this RFC (see the architecture section).

With `DummyAuth` wired, all of the checks that do exist are no-ops. Any caller who can reach the gRPC port can create, read, update, or delete any resource in any project namespace, on a control plane that also holds the credentials to create Ray/Spark workloads on compute clusters and, via the chart's static object-storage access keys (an access-key/secret pair mounted from a Kubernetes Secret; the minio client's IAM branch exists at `go/base/blobstore/minio/module.go` but no chart value reaches it), to touch object storage. There is no notion of "this user owns project A but not project B."

The gap is visible in the project's own proposal queue: the open Human Identity RFC (michelangelo-ai/enhancements#16) leads with replacing `DummyAuth`, and merged UI work (michelangelo-ai/michelangelo#1504, 2026-07-14) already ships a `UserRole` enum in the frontend core package for downstream authorization decisions -- scaffolding waiting on an enforcement story that does not exist yet. The public roadmap does not mention authentication or authorization at all as a capability; the closest reference is "Team ownership via OSS ownership model (CODEOWNERS)" under Project Management, which is a repository-governance concern, not a runtime one.

There is also a live doc/code split worth calling out explicitly. `docs/operator-guides/setup/authentication.md` (merged via PR #1039) already reads as a complete operator guide for this feature: it documents an `apiserver.auth.rbacEnabled` flag, an `apiserver.auth.oidc` block with `issuerUrl`, `clientId`, `usernameClaim`, and `groupsClaim`, an `apiserver.auth.sessionTokenExpiry` setting, worked walkthroughs for Okta/Google Workspace/Azure AD/Keycloak, and RoleBinding examples that grant a user or group access to a project namespace via a `viewer` or `editor` ClusterRole. None of this is backed by code today. `go/auth/auth.go` has no `rbacEnabled` field, no OIDC client, and no reference to Kubernetes RBAC. An operator who follows this guide today edits a ConfigMap key that `DummyAuth` never reads, restarts the API server per the guide's own instructions, and gets exactly the same allow-all behavior as before, with no error or warning that the configuration was ignored. The guide is not wrong about what the platform should do; it is just ahead of what the platform actually does.

## Motivation

Any organization installing this control plane for more than one team hits the multi-tenancy question in week one, and today the honest answer is "there isn't one." This is the specific gap that migrants from Flyte or Kubeflow notice immediately, since both ship real per-project RBAC out of the box. It also blocks every other trust-sensitive feature that depends on knowing who is calling: audit trails, per-project quota, approvals, personalization.

Two design choices matter for how this gets solved, and both are given by the existing shape of the codebase rather than invented here:

First, the seam should stay pluggable -- but the contract does not need to be bespoke. `k8s.io/apiserver`'s `authenticator` and `authorizer.Authorizer` interfaces are the ecosystem's standard shape for exactly this seam, and they are already direct dependencies of this repository. Adopting them makes pluggability strictly better than the current two-method `Auth` interface: an organization can implement `authorizer.Authorizer` in Go, or point a `SubjectAccessReview` webhook at a policy engine such as OPA and write no Go at all, and `union.New` composes multiple authorizers in a well-understood order. The right contribution is a generic OIDC-plus-Kubernetes-RBAC assembly of those upstream pieces that works with any OIDC-compliant issuer -- not a hardcoded dependency on one vendor, and not a second homegrown interface either.

Second, the `Action` enum in `go/auth/auth.go` (`Create`, `Get`, `Update`, `Delete`, `DeleteCollection`, `List`) already reads as a set of Kubernetes RBAC verbs with the casing changed, and `UserAuthorized`'s `namespace` parameter is already a Kubernetes namespace (`request.Cluster.Namespace`, i.e. the project). Nothing about the interface asks for a bespoke roles table. Building one anyway would mean asking every adopter to learn and administer a second permission system that duplicates what Kubernetes RBAC already does, when the interface's own shape points at reusing it directly. It would also fight the interface's evident purpose: `Auth` exists as a swappable seam with a single selection point precisely so a deployment can plug an organization-specific authorizer behind it, and a generic implementation should stay equally swappable, not become the only path.

Composing with existing Kubernetes RBAC also means operators inherit tooling they likely already use for every other resource in the same namespace: `kubectl auth can-i`, RBAC linting and visualization tools, and their existing GitOps process for RoleBindings. Nothing new to learn, nothing new to audit.

## Goals

- Replace the bespoke `Auth` contract with the ones `k8s.io/apiserver` already defines (both packages are already direct dependencies): authentication behind `authenticator.Token`, producing a `user.Info`; authorization behind `authorizer.Authorizer`, consuming an `authorizer.AttributesRecord` built from the validated identity plus the existing `(namespace, Action, resourceType)` triple and answered by a Kubernetes `SubjectAccessReview`. The allow-all default becomes `authorizerfactory.NewAlwaysAllowAuthorizer` plus a trivial always-authenticated token authenticator, preserving today's `DummyAuth` behavior exactly; organization-specific authorizers compose via `union.New`, in Go or as a `SubjectAccessReview` webhook (OPA and friends) with no Go at all.
- Reads enforced like writes: fix the code-generation gap so Get and List handlers perform the same authentication and authorization calls the mutating handlers already do. Under the default `DummyAuth` the regenerated checks are no-ops, so the fix is behavior-preserving for existing deployments and lands first.
- First-party callers keep working when enforcement is turned on: the same implementation accepts Kubernetes ServiceAccount tokens (validated via `TokenReview`) alongside human OIDC tokens. The worker's pipeline-execution RPCs authenticate as a **per-project runner ServiceAccount** (`system:serviceaccount:<project-ns>:michelangelo-runner`, via short-lived `TokenRequest` tokens) and are authorized through the same `SubjectAccessReview` path against namespace-scoped `RoleBinding`s -- deliberately not as a single platform-wide worker identity, which would let anyone who can trigger a run act with the worker's permissions instead of the project's (see the machine-identity section). Without a machine-identity answer, enabling the mode would halt every running pipeline (see Non-goals for what internal callers actually send today).
- Identity-provider-agnostic by construction: any OIDC-compliant issuer (Okta, Auth0, Azure AD, Google Workspace, Keycloak, Dex, or a homegrown IdP) works through standard issuer discovery and JWKS, with no code specific to any one provider.
- Authorization expressed entirely as ordinary Kubernetes `Role`/`ClusterRole`/`RoleBinding`/`ClusterRoleBinding` objects against the `michelangelo.api` resources, not a new permission model owned by this project.
- Fully additive and backward compatible: the default remains allow-all and behaviorally identical to today's `DummyAuth` (`mode: dummy`, now assembled from `NewAlwaysAllowAuthorizer` plus an always-authenticated token authenticator), and stays the documented sandbox/local-dev configuration. Enforcement is opt-in via config.
- Reuse the `*rest.Config` the API server already constructs (`go/base/config/config.go`'s `GetK8sConfig`, already fx-provided in `main.go`) for the `SubjectAccessReview` and `TokenReview` calls, rather than introducing a second Kubernetes credential path.
- Close the doc/code gap: bring `docs/operator-guides/setup/authentication.md` in line with what the shipped config actually does -- including shipping the `michelangelo-viewer`/`michelangelo-editor` ClusterRoles and updating the guide's RoleBinding examples to reference them. The guide's bare `viewer`/`editor` names are defined by nothing today: the chart's only ClusterRole is the platform's own, and Kubernetes' built-in `view`/`edit` aggregate roles do not cover `michelangelo.api` resources.
- No proto or CRD changes, and no new gRPC/REST surface. Two code-generation changes are required and in scope -- the Get/List auth block in `go/kubeproto/templates/crd_svc.tmpl`, and the switch of the emitted auth calls from the retired `Auth` methods to the new contracts, each with regeneration of `proto-go/api/v2/` -- but retiring `Auth` is an internal Go seam change; nothing on the wire and no proto message is touched.

## Non-goals

- No bespoke role or permission model (no built-in `admin`/`user` roles, no permissions table). Where a simpler role convention is wanted, it should be layered as a provisioning convenience on top of Kubernetes RBAC (see Open questions), not as a second authorization code path.
- No Web UI login flow, no PKCE or device-grant handling, no `oauth2-proxy` deployment, no `/api/v1/me` endpoint. This RFC's authentication and authorization implementation only validates a bearer token that is already present on a request; it does not care how the caller obtained it. See "Relationship to the Human Identity RFC" below for how this composes with the in-flight proposal that does own token acquisition.
- No session management inside the API server. `sessionTokenExpiry`, as named in the existing operator guide, is a concern for whatever sits in front of a browser session (an edge proxy, if one is deployed); the API server itself only checks each token's own `exp`/`nbf`/`iat` claims per request.
- No bundled identity provider. `mode: dummy` (allow-all) stays the zero-dependency sandbox default; this RFC does not ship a Dex or Keycloak subchart.
- No service-mesh or mTLS identity overhaul. Machine identity itself is deliberately in scope -- it has to be, because today's first-party callers send no credential at all. The worker's apiserver dial is at most server-authenticating TLS with nothing attached (`go/worker/config.go`; no code under `go/worker/` reads a ServiceAccount token or sets an authorization header). `mactl` sends a static `rpc-caller: grpcurl` metadata value over an insecure channel by default (`python/michelangelo/cli/mactl/mactl.py`, defaults in its `config.py`). The Web UI sends a static `Rpc-Caller: ma-studio` plus client-supplied `x-user-email`/`x-user-name` headers. Since the worker's pipeline activities call the auth-gated Create handlers (`go/worker/activities/ray/ray_activities.go`, `go/worker/activities/cachedoutput/cachedoutput_activities.go`), enforcement without a machine-identity answer would halt every running pipeline. This RFC therefore includes the minimal answer (ServiceAccount tokens via `TokenReview` -- see architecture) and excludes only the larger service-identity topics: mTLS/SPIFFE, workload identity federation, and credential-rotation policy. One honesty note: `authentication.md`'s "Service Authentication (Internal)" section claims the worker "uses its Kubernetes pod service account token" -- like the document's RBAC section, that describes intent, not shipped code.
- No per-project quota enforcement. That composes naturally with this work (a `RoleBinding` establishes who can act in a project; quota governs how much), but quota is a separate concern with its own natural substrate (Kueue `ClusterQueue`/`ResourceQuota`) and is out of scope here.
- Not a port of any organization-specific identity system. The seam stays a seam; an organization-specific authorizer (an org-graph or people-group system, say) remains a private implementation of `authorizer.Authorizer`, composed ahead of or behind the `SubjectAccessReview` authorizer via `union.New`, and is never part of this proposal.

## High-level architecture

The design retires the two-method `Auth` interface in favor of the contracts `k8s.io/apiserver` already defines for this seam -- `authenticator.Token` for authentication and `authorizer.Authorizer` for authorization -- and assembles the generic implementation from that package's own building blocks rather than hand-rolling them. Two narrow code-generation changes fall out, each behavior-preserving on its own. First, the generated Get and List handlers call nothing today, so the kubeproto handler template (`go/kubeproto/templates/crd_svc.tmpl`, which `templates.go` embeds) gains in its Get/List sections the same authentication-and-authorization block that its Create/Update/Delete/DeleteCollection sections already emit, followed by regeneration of `proto-go/api/v2/`; under the default allow-all wiring the new checks return allow unconditionally, so that regeneration is behavior-preserving and lands as its own PR ahead of everything else, against the existing interface. Second, the contract swap itself: the emitted block switches from `UserAuthenticated`/`UserAuthorized` to authenticate-then-authorize against the new contracts, with allow-all defaults keeping behavior identical.

```
Caller (mactl, UI via BFF/gateway, worker via per-project runner tokens, automation)
        |
        |  bearer token on the yarpc/gRPC call
        v
Generated handler (proto-go/api/v2/*.pb.kubeyarpc.go)
        |
        |-- authn: bearer from yarpc headers ---------------> authenticator.Token
        |     (yarpc.CallFromContext(ctx).Header("authorization"))
        |     tokenunion.New(
        |         OIDC token authenticator  - issuer discovery + cached JWKS
        |                                   - check iss, aud (any-of configured audiences),
        |                                     exp, nbf, iat (with clock-skew leeway)
        |                                   - usernameClaim/groupsClaim -> user.Info
        |         TokenReview authenticator - K8s ServiceAccount tokens (runner SAs)
        |                                   - identity system:serviceaccount:<ns>:<name>
        |     ), wrapped in a short-TTL token cache
        |
        |-- authz: authorizer.Authorizer.Authorize(ctx, ----> DelegatingAuthorizerConfig-built
        |     authorizer.AttributesRecord{                    SubjectAccessReview authorizer
        |         User: <user.Info from authn>,               - submits via existing *rest.Config's
        |         Verb: <Action mapped, e.g.                    clientset
        |             DeleteCollection -> "deletecollection">, - built-in short-TTL allow/deny
        |         APIGroup: "michelangelo.api",                 decision cache
        |         Resource: <kind mapped, e.g.                - org-specific authorizers compose
        |             "ModelFamily" -> "modelfamilies">,        ahead of it via union.New
        |         Namespace: ns, ResourceRequest: true})
        |     deny unless the decision is DecisionAllow
        v
Existing business logic (apiHandler.Create, hooks, audit log) -- unchanged
```

**Token extraction reuses an existing pattern.** `go/api/utils/utils.go`'s `GetHeaders(ctx)` already shows how to pull yarpc headers out of a handler's context via `yarpc.CallFromContext(ctx)`. A small helper uses the same mechanism to read the `authorization` header and hand the bearer token to `authenticator.Token` -- which is also why the token-shaped contract fits here better than `authenticator.Request`: these handlers have a yarpc context, not an `*http.Request`.

**One validation per request, by construction.** Today's `Auth` pair forces two independent interface calls per request with nothing threading identity from the first into the second. The upstream contracts dissolve that wart: the authenticator returns a `user.Info` once, and the handler passes it straight into the `AttributesRecord` the authorizer consumes. No duplicate token validation, no `ctx`-threading plumbing to design -- the chain is explicit in the contract types themselves.

**Resource and verb mapping.** The `resourceType` string that generated handlers already pass (`"Cluster"`, `"ModelFamily"`, `"EvaluationReport"`, etc.) maps to the CRD's plural, lowercase resource name already enumerated in `helm/michelangelo/templates/rbac/clusterrole.yaml` (`clusters`, `modelfamilies`, `evaluationreports`, ...) under the `michelangelo.api` API group. The `Action` enum's values already read as Kubernetes verbs with different casing (`Get` -> `get`, `DeleteCollection` -> `deletecollection`); the mapping is a straightforward lowercase conversion, not a redesign. Phase 1 ships this as a small static table; see Open questions on whether to derive it from the already-loaded `runtime.Scheme` instead, to avoid drift as new resource types are added.

**Reusing the existing Kubernetes client path.** `go/api/crd/gateway.go` already constructs an `apiextensionsclientset` from the same `*rest.Config` that `main.go` provides via `fx.Provide(baseconfig.GetK8sConfig)`. The `SubjectAccessReview` authorizer follows the identical pattern: a `kubernetes.Clientset` built from that config, its `AuthorizationV1().SubjectAccessReviews()` client handed to `authorizerfactory.DelegatingAuthorizerConfig`. No second kubeconfig, no second set of credentials, no new RBAC concept for the platform to invent -- just one more client built from a dependency that already exists in the fx graph.

**SubjectAccessReview cost is bounded by a decision cache.** Token validation is process-local, but each authorization call as described is a Kubernetes API round trip -- and with Get/List enforced, that is every RPC in the system, including the UI's polling reads. The standard answer is the one `kube-apiserver`'s own delegated-authorization stack uses: cache allow and deny decisions for a short TTL, keyed on `(user, groups, verb, resource, namespace)`. `k8s.io/apiserver` is already a direct dependency (`go/go.mod`, v0.31.14), and its delegating building blocks supply both the `SubjectAccessReview` call and the cache (`authorizerfactory.DelegatingAuthorizerConfig` with its allow/deny cache TTLs, and the token-cache equivalents on the authentication side) -- this RFC assembles them rather than hand-rolling either. Default TTLs of 10s/10s make revocation lag bounded and explicit; the tradeoff appears in the failure-modes table.

**Machine identity: per-project runner ServiceAccounts, not one platform-wide worker identity.** Today no first-party caller sends any credential (see Non-goals for the per-caller inventory), and the worker's pipeline activities perform mutating RPCs against these very handlers -- `CreateRayCluster` and `CreateRayJob` from `go/worker/activities/ray/ray_activities.go`, `CreateCachedOutput` from `go/worker/activities/cachedoutput/cachedoutput_activities.go` -- so an authorization mode that only understands human OIDC tokens would stop every pipeline at its first activity callback.

A single platform-wide worker ServiceAccount, authorized by one chart-shipped `RoleBinding` granting it the verbs its activities need, is a confused deputy: the activities take their namespace from the request payload, so the one binding would have to span every tenant namespace, and anyone who can trigger a run would act with the worker's near-platform-wide permissions instead of the project's -- undoing the per-namespace isolation the rest of the design exists to provide. The ecosystem is consistent on how to avoid this: **namespaced work runs as a namespace-local identity, not as the platform's.** Argo Workflows executes each workflow under a ServiceAccount in the workflow's own namespace; Kubeflow's multi-user pipelines run as the per-profile-namespace `default-editor` SA; Tekton takes a per-`PipelineRun` ServiceAccount; Flyte scopes execution identity to per-project/domain namespaces.

This RFC adopts the same pattern. Each project namespace gets a `michelangelo-runner` ServiceAccount and a `RoleBinding` granting it, in that namespace only, the verbs pipeline activities need. For an activity targeting project namespace `ns`, the worker's outbound middleware (`go/worker/config.go` dispatcher -- the only worker-side change, off by default until its Phase 2 grants exist; see rollout) requests a short-lived token for `michelangelo-runner` in `ns` via the `TokenRequest` API and attaches it as the bearer header. `TokenReview` then yields `system:serviceaccount:<ns>:michelangelo-runner`, and the identical `SubjectAccessReview` path authorizes it against that namespace's binding (SA tokens are JWTs, but the cluster -- not this RFC's JWKS path -- is their authority). The worker's own ServiceAccount needs exactly one narrow cluster-wide grant: `create` on `serviceaccounts/token` restricted with `resourceNames: ["michelangelo-runner"]`. The blast radius follows: a caller who can trigger a run in project A gets, transitively, at most the runner's rights in A -- never the worker's rights elsewhere. `mactl` and the Web UI carry human tokens instead, acquired per the Human Identity RFC (see below).

**Get/List authorization semantics.** Get authorizes exactly like the write verbs: `(request.Namespace, get, resource)` -- the request message already carries the namespace. List is the one place the triple is not obviously namespaced: `ListClusterRequest.Namespace` may be empty, meaning a cluster-wide list, which is what the UI's global views issue today. A `SubjectAccessReview` with an empty namespace asks "can this user list this resource across the whole cluster," and namespace-scoped users will rightly be denied. Phase 1 takes the simple, safe semantics: an empty-namespace List requires the cluster-wide grant and is otherwise denied with a distinct reason, and the operator guide steers namespace-scoped users to namespaced list calls. The friendlier alternative -- fan out per-namespace `SubjectAccessReview`s and filter results to permitted namespaces, the pattern Kubernetes dashboards use -- preserves global views for scoped users at the cost of N checks plus result filtering, and is posed as an open question rather than assumed.

**A useful side effect.** Because authorization decisions are ordinary `SubjectAccessReview`s against real `RoleBinding`/`ClusterRoleBinding` objects, an operator can validate a policy before wiring up OIDC at all, using nothing but `kubectl`: `kubectl auth can-i create clusters.michelangelo.api -n team-a --as=alice@example.com`. One correction to the published guide is required to get there, though: `authentication.md`'s RoleBinding examples reference `viewer` and `editor` ClusterRoles that nothing defines -- the chart's only ClusterRole is the platform's own, and Kubernetes' built-in `view`/`edit` aggregate roles do not cover `michelangelo.api` resources, so an operator following the guide today creates a RoleBinding that grants nothing. Shipping those ClusterRoles (`michelangelo-viewer` and `michelangelo-editor`, with the guide updated to match) is therefore a required Phase 2 deliverable of this RFC, not a convenience; with them in place the guide's examples become literally correct rather than needing to be rewritten from scratch.

### Relationship to the Human Identity RFC (michelangelo-ai/enhancements#16)

An open PR in the enhancements repo, "Human Identity" (#16, @craigmarker -- open and out of draft, though with no review discussion on it yet), also proposes replacing `DummyAuth`. It is worth being precise about where the two overlap and where they do not, since both cannot simply land independently without reconciliation -- and since no reconciliation thread exists yet, starting one is itself an action item of this RFC.

\#16's scope is how a human obtains a token in the first place: `Authorization Code + PKCE` for the Web UI via `oauth2-proxy` at the ingress, `Device Authorization Grant` for `mactl`, a `/api/v1/me` BFF endpoint, and a bundled Dex/Keycloak option for sandbox use. It explicitly defers service-to-service auth to a separate RFC. On the apiserver side, #16 proposes "replace `DummyAuth` with a real OIDC token validator" plus "a minimal but real RBAC model (`admin`/`user` roles, derived from OIDC `groups`)."

That apiserver-side bullet is exactly this RFC's scope, and the two proposals currently disagree on the authorization model: #16 sketches a bespoke two-role global model, while this RFC maps to per-namespace Kubernetes RBAC. They agree on everything upstream of that: both assume an OIDC-compliant issuer, both want a real token validator behind `Auth`, both want `DummyAuth` to remain the default. The disagreement is not hypothetical or idle, either -- #16's role model is already growing frontend roots: PR michelangelo-ai/michelangelo#1504 (merged 2026-07-14) shipped a `UserRole` enum in the UI core package for downstream authorization decisions, consumed by the navigation bar. Code accretes around whichever model exists first, which raises the cost of deferring the reconciliation.

The proposed reconciliation is a layering, not a merge of code: #16 continues to own how a caller acquires a token (browser login, device grant, bundled IdP for sandboxes) and this RFC owns what the apiserver does once a bearer token, from any source, arrives -- validate it, and turn `(identity, namespace, Action, resourceType)` into an allow/deny via Kubernetes RBAC. `#16`'s two-role convenience does not have to be abandoned; it can be layered over the ClusterRoles this RFC already ships (`user` mapping to `michelangelo-editor` bindings in the relevant namespaces, `admin` to a cluster-wide `michelangelo-admin` if wanted) with an optional Helm-driven step that provisions `RoleBinding`s from a `groupRoleMapping`, sitting on top of this RFC's `SubjectAccessReview` authorizer rather than beside it as a second code path.

One concrete integration detail worth surfacing now: #16's Web UI path does not put a bearer JWT on the wire to the apiserver by default -- `oauth2-proxy` is described as injecting `X-Auth-Request-User`/`X-Auth-Request-Email`/`X-Auth-Request-Groups` headers for its own BFF endpoint to read. For this RFC's bearer-token validator to also cover Web UI traffic without a second, header-trust authentication mode, `oauth2-proxy` needs to be configured to forward the original token (its `--pass-authorization-header` option, which forwards the ID token; if access tokens were validated instead, the flag and header differ -- `--pass-access-token` / `X-Forwarded-Access-Token`) so the same code path handles UI, CLI, and any direct API caller uniformly. Three mechanical consequences follow:

- **Audience validation must accept a set, not a single client ID.** Under #16, UI tokens would carry `oauth2-proxy`'s client ID while `mactl`'s device-grant tokens carry `mactl`'s; a single-`clientId` check rejects one of them. The validator therefore takes an `audiences` list, matched any-of.
- **The shipped Envoy CORS config blocks the header this design reads.** `helm/michelangelo/templates/core/envoy-configmap.yaml`'s `allow_headers` list does not include `authorization` (it does include the spoofable `x-user-email`/`x-user-name`), so a browser's preflight would strip the bearer token before it ever reached the validator. Adding `authorization` to that list is a one-line Phase 2 change and is needed regardless of which proposal lands first.
- **Until #16's token acquisition ships, the Web UI cannot use this mode at all.** The UI has no way to obtain or attach a token today, so a deployment that enables `k8s-rbac` mode before #16 lands has a fully-enforced API and a UI that cannot authenticate to it (reads included, once Get/List enforcement is in). The rollout section states this ordering constraint plainly rather than leaving it to be discovered.

Related hygiene, one sentence: the UI's client-supplied `x-user-email`/`x-user-name` headers are today trusted for actor attribution (`javascript/packages/rpc/handlers.ts` stamps `spec.actor` on created PipelineRuns from them); once validated identities exist, attribution should derive from the token server-side, since a spoofable audit field is arguably worse than none.

## APIs and CRDs

No new CRDs and no proto changes. Beyond the two template changes -- which regenerate `proto-go/api/v2/` but change no message or wire format -- everything lives behind the `authenticator.Token`/`authorizer.Authorizer` seam.

The new Go surface is deliberately thin assembly over `k8s.io/apiserver`, plus the one mapping this codebase owns:

```
go/auth/            // retires the Auth interface; provides the factory below, the trivial
                    // always-authenticated token authenticator for mode "dummy", and the
                    // (Action, resourceType) -> AttributesRecord mapping helper
go/auth/oidc/       // assembly: OIDC token authenticator (issuer discovery + cached JWKS +
                    // claims -> user.Info), tokenunion.New with the TokenReview-backed
                    // authenticator, wrapped in the upstream token cache
go/auth/k8srbac/    // assembly: authorizerfactory.DelegatingAuthorizerConfig -> SubjectAccessReview
                    // authorizer with its built-in allow/deny decision cache
```

Module wiring: one always-wired factory whose constructor switches on config -- the selection happens inside the provider, not by conditionally including fx modules (an `fx.Option` returned from a provider is just an inert value; fx never applies it, and config is not in the graph before the app is built anyway). This is the same factory-by-config shape the operator guide's Option 2 sketches for the scheduler and the companion Kueue RFC adopts:

```go
// go/auth/module.go
type AuthBundle struct {
    fx.Out
    Authenticator authenticator.Token
    Authorizer    authorizer.Authorizer
}

var AuthModule = fx.Options(
    fx.Provide(func(provider config.Provider, k8sConfig *rest.Config) (AuthBundle, error) {
        var cfg AuthConfig
        if err := provider.Get("apiserver.auth").Populate(&cfg); err != nil {
            return AuthBundle{}, err
        }
        switch cfg.Mode {
        case "", "dummy":
            return AuthBundle{
                Authenticator: alwaysAuthenticated{}, // ~5 lines; the one piece with no upstream equivalent
                Authorizer:    authorizerfactory.NewAlwaysAllowAuthorizer(),
            }, nil
        case "k8s-rbac":
            authn, err := oidc.NewTokenAuthenticator(cfg.OIDC, k8sConfig) // tokenunion(OIDC, TokenReview) + cache
            if err != nil {
                return AuthBundle{}, err
            }
            authz, err := k8srbac.NewSARAuthorizer(k8sConfig, cfg.SARCache) // DelegatingAuthorizerConfig
            if err != nil {
                return AuthBundle{}, err
            }
            return AuthBundle{Authenticator: authn, Authorizer: authz}, nil
        default:
            return AuthBundle{}, fmt.Errorf("unknown apiserver.auth.mode %q", cfg.Mode)
        }
    }),
)
```

`main.go`'s only change is swapping `auth.DummyAuthModule` for `auth.AuthModule`; whenever the mode is unset or `"dummy"`, behavior is exactly today's allow-all.

Configuration (consumed by the API server's existing `config.Provider`, following the same `provider.Get(key).Populate(&struct)` pattern as `GetMySQLConfig`/`GetMetadataStorageConfig` in `go/base/config/config.go`):

```yaml
apiserver:
  auth:
    mode: k8s-rbac          # "dummy" (default) | "k8s-rbac"
    oidc:
      issuerUrl: https://accounts.your-idp.com
      audiences: [michelangelo]   # any-of; list several when UI and CLI use different client IDs
      usernameClaim: email        # matches the naming already used in authentication.md
      groupsClaim: groups
      clockSkewLeeway: 60s
      jwksCacheTTL: 15m
    serviceAccounts:
      enabled: true               # accept K8s ServiceAccount tokens via TokenReview (first-party callers)
    sarCache:
      allowTTL: 10s               # 0 disables caching
      denyTTL: 10s
```

This keeps the `apiserver.auth.*` nesting and the `oidc.{issuerUrl,usernameClaim,groupsClaim}` naming that `docs/operator-guides/setup/authentication.md` already publishes, since that is the operator-facing contract already in the repository, rather than introducing a third naming scheme. There are two deliberate divergences, both corrected in the doc in Phase 2: the doc's single `clientId` becomes the `audiences` list for the multi-client reason above, and its boolean `rbacEnabled` becomes `mode` -- an enum, since more than one real implementation can now sit behind the flag. The `apiserver.`-prefixed nested key is the existing convention -- `"apiserver.yarpc"` (`go/cmd/apiserver/config.go`) and `"apiserver.crdSync"` (`go/api/crd/sync.go`) are real keys read the same way -- so `apiserver.auth` extends a live pattern rather than inventing one. One cautionary precedent nearby: the chart's rendered apiserver ConfigMap nests `k8s:` under `apiserver:` while the Go side reads a top-level `k8s` key (`GetK8sConfig` in `go/base/config/config.go`), a pre-existing chart/code mismatch this RFC must not replicate -- Phase 2's e2e therefore asserts that the rendered ConfigMap key actually populates `AuthConfig`. Reconciling all of this with #16's proposed top-level `auth.*` Helm keys is an open question below.

Helm values addition, following the existing `apiserver:` block in `helm/michelangelo/values.yaml`:

```yaml
apiserver:
  auth:
    mode: dummy   # default unchanged; set to "k8s-rbac" to enable
    oidc:
      issuerUrl: ""
      audiences: []
      usernameClaim: email
      groupsClaim: groups
    serviceAccounts:
      enabled: true
```

Helm chart RBAC addition, alongside the existing rules in `helm/michelangelo/templates/rbac/clusterrole.yaml` -- the API server's own ServiceAccount needs permission to create `SubjectAccessReview`s (the authorization check) and `TokenReview`s (the ServiceAccount-token authentication path):

```yaml
  # Required for the SubjectAccessReview authorizer to check callers' permissions
  - apiGroups: ["authorization.k8s.io"]
    resources: ["subjectaccessreviews"]
    verbs: ["create"]
  # Required to validate first-party callers' ServiceAccount tokens via TokenReview
  - apiGroups: ["authentication.k8s.io"]
    resources: ["tokenreviews"]
    verbs: ["create"]
```

The chart also gains, in the same phase: the per-project-namespace `michelangelo-runner` ServiceAccount and `RoleBinding` (see machine identity -- provisioning lifecycle is an open question below); the worker's narrow token-mint grant (`create` on `serviceaccounts/token`, `resourceNames: ["michelangelo-runner"]`); the `michelangelo-viewer`/`michelangelo-editor` ClusterRoles the operator guide's examples depend on; and `authorization` appended to the Envoy CORS `allow_headers` list.

Operator-facing `RoleBinding` example (the shape `authentication.md` already documents, now actually enforced -- with the roleRef pointing at a ClusterRole this RFC actually ships):

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: alice-reader
  namespace: ml-team-project
subjects:
- kind: User
  name: alice@your-company.com
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: michelangelo-viewer   # shipped by the chart in Phase 2; the guide's bare "viewer" exists nowhere today
  apiGroup: rbac.authorization.k8s.io
```

## Alternatives considered

### Alternative A: Bespoke in-process RBAC (a roles/permissions table, as sketched in RFC #16)

**Pros:** Simple mental model for a first cut; two roles (`admin`/`user`) cover a lot of early deployments; no dependency on the Kubernetes authorization API; keeps working even if the `SubjectAccessReview` call path is briefly unavailable, since the check would be local.

**Cons:** Reinvents a permission model Kubernetes already ships and that every operator running this on Kubernetes already administers for other resources. Two global roles do not express the actual multi-tenancy requirement -- "team A can act on team A's namespace, not team B's" -- without still building a per-project layer on top, at which point most of the value of "simple" is gone. It also forecloses reusing existing K8s RBAC tooling (`kubectl auth can-i`, policy linting, existing GitOps flows for RoleBindings) that operators already have.

**Why not chosen:** The design goal here is composing with an existing, well-understood primitive rather than owning a second one. That does not have to mean discarding #16's two-role convenience entirely -- see "Relationship to the Human Identity RFC" above for a path that keeps it as a provisioning convention over this RFC's authorizer instead of a parallel authorization implementation.

### Alternative B: Gateway or service-mesh-level auth (oauth2-proxy in front of Envoy, or Envoy ext_authz / a mesh AuthorizationPolicy)

**Pros:** Keeps authentication and coarse authorization out of application code entirely; battle-tested; is in fact what #16 proposes for the Web UI's login flow specifically; would not need per-language reimplementation if more API surfaces appear later; can reject unauthenticated traffic before it costs the apiserver anything.

**Cons:** The chart's shipped Envoy configuration (`helm/michelangelo/templates/core/envoy-configmap.yaml`) today is purely a grpc-web/JSON transcoder for the Web UI, with no auth filter configured, and Envoy only fronts the Web UI's grpc-web path -- `mactl` and any other direct caller talk gRPC/yarpc straight to the apiserver's port, bypassing Envoy entirely (`mactl` opens its channel directly against the configured address, `python/michelangelo/cli/mactl/mactl.py`; the network guide's topology diagram shows the same direct paths, though it never mentions `mactl` by name). A gateway can answer "is this request authenticated," but the actual authorization decision needs the `(namespace, Action, resourceType)` triple that only exists once a generated handler has parsed the request -- pushing authorization to the gateway would mean duplicating that parsing at the edge, or falling back to coarse path-based rules that cannot express "can this user delete a Cluster in this specific project."

**Why not chosen as a replacement:** This is a good complementary layer, and it is exactly the layer #16 proposes for the Web UI's login UX. It is not a substitute for request-scoped, resource-shaped authorization inside the apiserver, which is where the `Auth` interface already lives and where the triple it needs is already computed for free. This RFC's validator also has to work standalone for callers that never transit a gateway, such as `mactl` or any future automation client.

### Alternative C: Kubernetes user impersonation (`Impersonate-User`/`Impersonate-Group`) instead of SubjectAccessReview

**Pros:** Lets the Kubernetes API server do both the authorization check and the actual read/write in a single round trip, under its normal RBAC enforcement; no separate authorization call.

**Cons:** Requires granting the apiserver's own ServiceAccount the `impersonate` verb on `users` and `groups`, a much broader and riskier grant than `create` on `subjectaccessreviews` -- a compromised or buggy apiserver process could act as any user in the cluster. More fundamentally, the apiserver does not proxy Kubernetes object CRUD 1:1 today; `clusterServiceHandler.CreateCluster` runs its own validation, `BeforeCreate`/`OnCreateSuccess` API hooks, metadata-storage writes, and audit logging around the underlying Kubernetes write, all currently executed as the control plane's own service identity. Switching the execution identity to the impersonated user would mean re-plumbing every one of those code paths to work correctly under an arbitrary caller's Kubernetes permissions, a substantially larger change than adding a permission check ahead of business logic that keeps running as today's service identity.

**Why not chosen:** `SubjectAccessReview` answers exactly the question the authorizer needs answered -- would this identity be allowed to do this -- without changing which identity actually performs the underlying write. Impersonation conflates the authorization question with execution identity and asks for a far riskier grant to get there.

### Alternative D: Leave DummyAuth as the only supported implementation

**Pros:** Zero code; consistent with "the interface exists so operators can plug in their own thing."

**Cons:** Leaves `authentication.md` permanently describing behavior the platform does not have, and leaves every adopter without an in-house identity team to either build OIDC+RBAC from scratch or run with no access control at all -- the very gap the in-flight Human Identity RFC exists to close, so the demand is already visible in the project's own proposal queue.

**Why not chosen:** An interface is valuable because it is pluggable, but a control plane with only an allow-all implementation on day one is not usable for shared deployments. Shipping one correct, generic, opt-in implementation that covers the common case (standard OIDC issuer, standard Kubernetes RBAC) does not reduce pluggability for organizations with more specific identity systems; it just means the default path is not "no security" either.

## Open questions

- [ ] Reconciling with michelangelo-ai/enhancements#16 (Human Identity, @craigmarker; open and out of draft, with no review discussion yet -- so the reconciliation thread proposed here does not exist and starting it is the first action item): both proposals replace `DummyAuth`, and they currently specify different authorization models (per-namespace Kubernetes RBAC here vs. a global `admin`/`user` pair there), while merged UI work (michelangelo#1504's `UserRole` enum) is already building on #16's model. Recommend #16 delegate its apiserver-side "replace DummyAuth" and RBAC bullets to this RFC's authenticator/authorizer assembly, keeping #16 scoped to token acquisition (browser PKCE flow, device grant, bundled Dex for sandboxes). Needs sign-off from both authors and a maintainer before either lands its DummyAuth-replacing pieces.
- [ ] If #16's `oauth2-proxy` path ships as specified (injecting `X-Auth-Request-*` headers rather than forwarding the original bearer token), should `oauth2-proxy` be configured to pass through the raw token so this RFC's validator covers Web UI traffic through the same code path as `mactl` and direct API callers, or does the apiserver need a second, header-trust authentication mode? The former keeps one validation path; the latter is more common practice for gateway-fronted services but widens the trust boundary to "whatever is between the gateway and the apiserver."
- [ ] Provisioning convenience: should #16's `admin`/`user` convenience be layered as an optional Helm-driven `groupRoleMapping` that provisions `RoleBinding`s over the `michelangelo-viewer`/`michelangelo-editor` ClusterRoles this RFC ships, so both proposals converge on the single `SubjectAccessReview` enforcement path?
- [ ] Configuration naming: this RFC follows the `apiserver.auth.*` nesting already published (but unimplemented) in `authentication.md`, which also matches the real `apiserver.yarpc`/`apiserver.crdSync` key convention. #16 proposes a separate, top-level `auth.{enabled,oidc.*,oauth2Proxy.*,rbac.*}` Helm namespace. These need to converge before both merge; which one wins, and does `authentication.md` get corrected in place or superseded? Related: should the pre-existing chart/code `k8s:` nesting mismatch (chart nests it under `apiserver:`, Go reads top-level `k8s`) be fixed in this series or tracked separately?
- [ ] List semantics for namespace-scoped users: Phase 1 denies empty-namespace (cluster-wide) List for anyone without the cluster-wide grant, which breaks the UI's current global views for scoped users. Should Phase 2 move to per-namespace `SubjectAccessReview` fan-out with result filtering (the Kubernetes-dashboard pattern), accepting N-fold SAR volume and its interaction with decision-cache sizing, or is deny-plus-namespaced-calls the durable answer?
- [ ] Should the `resourceType`-to-CRD-plural mapping (for example `"ModelFamily"` -> `"modelfamilies"`) be a hand-maintained static table, or derived at startup from the already-loaded `runtime.Scheme`/CRD registrations in `go/api/crd`, to avoid silent drift when new resource types are added?
- [ ] Runner ServiceAccount provisioning lifecycle: who creates the `michelangelo-runner` SA and `RoleBinding` when a project namespace comes into being -- static Helm templating over a known project list, or the control plane's own project-creation path (the natural owner, since it already creates the namespace)? The latter gives every new project working pipelines with no operator step, at the cost of the platform managing RBAC objects; the guide documents whichever is chosen.
- [ ] Should the OIDC token authenticator reuse `k8s.io/apiserver/plugin/pkg/authenticator/token/oidc` (kube-apiserver's own, already in the dependency tree) rather than assembling one over a standalone OIDC library? Reuse maximizes consistency with the rest of the adopted stack; the plugin's configuration surface is shaped for kube-apiserver's flags, so a thin adaptation layer is needed either way.
- [ ] Should JWKS/issuer-discovery results be cached per apiserver replica (simplest, some duplicate fetch load across replicas) or in a shared cache (adds an operational dependency for a comparatively small savings)? Proposal: per-replica in-memory caching for v1. Same question, same proposed answer, for the SubjectAccessReview decision cache.
- [ ] What is the right maximum-staleness window for a cached JWKS before the apiserver fails closed on an unreachable issuer, and should that window be configurable per deployment?

## Rollout strategy

**Phase 1 (alpha).** Land in this order, each PR independently reviewable:
1. The Get/List enforcement fix: add the missing authentication/authorization block to the Get and List sections of `go/kubeproto/templates/crd_svc.tmpl` and regenerate `proto-go/api/v2/`. Behavior-preserving under the default `DummyAuth` (the checks return allow unconditionally), verified by a golden-file test asserting the write-verb handlers are byte-identical and Get/List gained exactly the shared block. Independently a correctness fix; nothing else in this series is honest without it, so it lands first, against the existing interface.
2. The contract swap: retire `Auth`/`DummyAuth` in favor of `authenticator.Token` + `authorizer.Authorizer` with the allow-all defaults; the template's emitted block switches to authenticate-then-authorize via an `AttributesRecord`; regenerate. Behavior-preserving (allow-all before and after), verified the same golden-file way plus an end-to-end assertion that every handler still allows everything under `mode: dummy`.
3. `go/auth/oidc`: the OIDC token authenticator assembly -- issuer discovery, JWKS caching, claims-to-`user.Info` with multi-audience support -- with full unit-test coverage against a local JWKS fixture.
4. `go/auth/k8srbac` + the `TokenReview` path: the `DelegatingAuthorizerConfig` SubjectAccessReview authorizer, the verb/resource attribute mapping, and `tokenunion` with the TokenReview-backed authenticator, unit-tested against a fake clientset.
5. The `AuthModule` factory and `apiserver.auth.*` config plumbing, default `dummy`, with a regression test asserting behavior is unchanged when the mode is unset or `"dummy"`.
6. The worker's outbound bearer-token middleware (`go/worker/config.go` dispatcher), minting and attaching the per-project `michelangelo-runner` token via `TokenRequest`. Shipped off by default behind a worker config flag, since the runner ServiceAccounts and the worker's token-mint grant are Phase 2 chart deliverables; with the flag off the dispatcher attaches nothing and worker traffic is byte-identical to today's.

No Helm changes are required to consume this phase since the feature is opt-in and off by default. No behavior change for any existing installation.

**Phase 2 (beta).** Add the `apiserver.auth.*` Helm values; both RBAC rule additions (`subjectaccessreviews` and `tokenreviews`) to `helm/michelangelo/templates/rbac/clusterrole.yaml`; the per-project `michelangelo-runner` ServiceAccount/`RoleBinding` machinery and the worker's narrow token-mint grant; the `michelangelo-viewer`/`michelangelo-editor` ClusterRoles the operator guide's examples require; `authorization` appended to the Envoy CORS `allow_headers`; and a startup self-check (submit a canary `SubjectAccessReview` and `TokenReview` at boot and log clearly if the apiserver's own ServiceAccount lacks permission, rather than surfacing that as a confusing per-request denial later). Add envtest-level tests proving real `RoleBinding` grants and denials flow through end to end, and a k3d e2e exercising a full OIDC-issuer-to-SubjectAccessReview path plus a worker-minted runner-token round trip (including the cross-namespace denial: project A's runner token must not act in project B), and asserting the rendered ConfigMap's `apiserver.auth` key actually populates `AuthConfig`. Rewrite `docs/operator-guides/setup/authentication.md` to match what actually ships -- including the corrected role names, the config-naming resolution, a rewritten "Service Authentication (Internal)" section describing the real mechanism, and a plainly-stated ordering constraint: until the Human Identity RFC's token acquisition ships, enabling `k8s-rbac` mode leaves the Web UI unable to authenticate (direct API callers and the worker are unaffected -- they attach their own tokens). Coordinate the open items with #16's author before either RFC's DummyAuth-replacing code merges.

**Phase 3 (GA).** Document `apiserver.auth.mode: k8s-rbac` as the recommended production configuration in the operator guide, with worked examples per common IdP (reusing the walkthroughs `authentication.md` already has for Okta/Google/Azure AD/Keycloak, now accurate). `DummyAuth` remains the default and is never removed; it stays the documented sandbox and local-development configuration indefinitely, per this design's backward-compatibility constraint.

**Rollback.** Setting `apiserver.auth.mode` back to `dummy` (or removing the key) and redeploying immediately restores today's allow-all behavior. No data migration is involved in either direction: authorization decisions are computed at request time from live JWTs and live `RoleBinding`/`ClusterRoleBinding` objects, and this feature persists nothing of its own.

**Failure modes and mitigations** (exercised in Phase 2 e2e where feasible):

- **Token expired.** Authentication fails and the handler's short-circuit applies before any authorization or business logic runs; the `metric.Counter("unauthenticated").Inc(1)` pattern already present in every generated handler carries over to the regenerated block, covering observability with no new code.
- **Issuer or JWKS endpoint unreachable.** Serve the last-known-good JWKS from cache with a background refresh and jittered retry; do not fail every in-flight request on a single failed refresh attempt. Fail closed (deny) only once the cache exceeds a configurable maximum staleness, and emit a distinct metric/log line (for example `jwks_refresh_failed`) well before that cutover so operators get a warning, not just an outage.
- **Clock skew between the apiserver and the issuer.** Apply a small, configurable leeway (default suggestion: 60 seconds) to `exp`/`nbf`/`iat` comparisons, consistent with standard JWT validation library behavior. Do not disable expiry checking to work around skew.
- **The SubjectAccessReview call itself fails** (Kubernetes API server unreachable, webhook timeout). Fail closed (deny), with a distinct error/metric from an explicit RBAC denial, so operators can tell "no RoleBinding exists" apart from "the Kubernetes API is down" in their alerting.
- **Malformed or missing bearer token.** Authentication fails before the authorizer is ever invoked, matching the short-circuit already present in every generated handler today.
- **Valid token, missing or empty groups/username claim.** Treated as authenticated but with no group memberships; the `SubjectAccessReview` is submitted as such and Kubernetes RBAC's own default-deny handles the rest with no special-case code required here.
- **Apiserver's own ServiceAccount cannot create SubjectAccessReviews or TokenReviews.** This is an install-time misconfiguration, not a per-request failure mode; caught by the Phase 2 startup self-check and documented as a Helm-chart RBAC prerequisite rather than surfacing as an opaque per-request 403 later.
- **A project namespace is missing its `michelangelo-runner` ServiceAccount or RoleBinding.** The worker's `TokenRequest` (or the subsequent `SubjectAccessReview`) fails for pipelines in that project only, surfaced as a distinct activity error naming the missing object; other projects are unaffected. Provisioning belongs to project creation (see the open question on lifecycle ownership).
- **A revoked RoleBinding is honored for up to the allow-cache TTL.** The SubjectAccessReview decision cache (default 10s allow / 10s deny) trades a bounded revocation lag for not making every RPC a Kubernetes API round trip. The window is configurable, `0` disables caching, and the tradeoff is documented rather than silent -- this is the same tradeoff `kube-apiserver`'s own delegated authorizers make.
- **Cluster-wide List from a namespace-scoped user.** Denied by design in Phase 1 (empty namespace = cluster-scoped check), with a deny reason distinct from "no RoleBinding in the named namespace" so operators and UI developers can tell the two apart. The per-namespace fan-out alternative is an open question.
- **`k8s-rbac` mode enabled before the Human Identity RFC's token acquisition exists.** The Web UI has no way to obtain or attach a token and fails authentication on every call (reads included). This is an ordering constraint documented in the operator guide, not a bug: the worker authenticates with its ServiceAccount token, and direct API callers can attach their own bearer tokens, so headless deployments can enable enforcement early; UI-dependent deployments should wait for #16 or front the UI with their own token-forwarding proxy.

**Testing plan.**

- Golden-file tests on both codegen changes: for the Get/List fix, regenerated write-verb handlers byte-identical and Get/List handlers gained exactly the shared auth block; for the contract swap, only the auth block changed in every handler, with an end-to-end allow-all assertion before and after.
- Unit tests for OIDC validation covering valid, expired, wrong-issuer, wrong-audience (including the multi-audience any-of semantics), bad-signature, and clock-skew cases against a local JWKS fixture, plus the ServiceAccount-token branch (valid SA token; TokenReview API unavailable -> fail closed).
- Unit tests for the attribute mapping (verb and resource), the empty-namespace List semantics, and the decision cache (TTL honored, allow and deny cached independently, `0` disables) against a fake Kubernetes clientset.
- Envtest-level tests proving that real `RoleBinding`/`ClusterRoleBinding` objects produce the expected allow/deny outcomes end to end, since envtest runs a real API server's authorization stack, unlike a fake clientset -- including the runner-token flow: mint via `TokenRequest`, authorize in-namespace, deny cross-namespace.
- A k3d e2e wiring a test-only OIDC issuer through to a live `SubjectAccessReview` decision, plus a worker-style runner-token call.
- A regression test asserting allow-all behavior is unchanged when `apiserver.auth.mode` is unset or `"dummy"`.

## References

- Code anchors: `go/auth/auth.go` (the `Auth` interface, `Action` enum, and `DummyAuth`); `go/cmd/apiserver/main.go:38` (`auth.DummyAuthModule`, the only auth module wired today); `proto-go/api/v2/cluster.pb.kubeyarpc.go` (representative generated handlers: `UserAuthenticated`/`UserAuthorized` at lines 195/204 in `CreateCluster` and equivalents in Update/Delete/DeleteCollection; `GetCluster` and `ListCluster` with no auth calls); `go/kubeproto/templates/crd_svc.tmpl` (the handler template whose Get/List sections omit the auth block; embedded by `templates.go`); `go/api/utils/utils.go` (`GetHeaders`, the existing `yarpc.CallFromContext(ctx)` pattern this RFC's token extraction reuses); `go/base/config/config.go` (`GetK8sConfig`, the `*rest.Config` this RFC's `SubjectAccessReview`/`TokenReview` clients reuse); `go/api/crd/gateway.go` (existing precedent for building a Kubernetes clientset from that same `*rest.Config`); `go/cmd/apiserver/config.go` and `go/api/crd/sync.go` (`apiserver.yarpc` / `apiserver.crdSync`, the nested config-key convention `apiserver.auth` extends); `go/worker/config.go` (the worker's apiserver dial: optional server-auth TLS, no credential attached); `go/worker/activities/ray/ray_activities.go` and `go/worker/activities/cachedoutput/cachedoutput_activities.go` (the worker's mutating RPCs that make machine identity mandatory); `python/michelangelo/cli/mactl/mactl.py` (direct-dial CLI, insecure channel and static `rpc-caller` metadata by default); `javascript/packages/rpc/create-fetch-transport.ts` and `javascript/packages/core/providers/user-provider/use-user-request-headers.ts` (the UI's static caller and client-supplied identity headers), `javascript/packages/rpc/handlers.ts` (actor attribution trusting those headers); `helm/michelangelo/templates/rbac/clusterrole.yaml` (the apiserver ServiceAccount's existing RBAC baseline, extended here; also the chart's only ClusterRole today); `helm/michelangelo/templates/core/envoy-configmap.yaml` (grpc-web transcoder only, no auth filter, and a CORS `allow_headers` list without `authorization`); `go/go.mod` (`k8s.io/apiserver` and `k8s.io/client-go` v0.31.14, already direct dependencies -- the source of the authenticator/authorizer contracts and delegating building blocks this design adopts).
- Docs: `docs/operator-guides/setup/authentication.md` (the existing, currently aspirational operator guide this RFC makes real -- both its RBAC section and its "Service Authentication (Internal)" section describe unshipped behavior; added by PR #1039); `docs/getting-started/roadmap.md` (auth is not a roadmap line item; the only related mention is "Team ownership via OSS ownership model (CODEOWNERS)" under Project Management).
- Related: michelangelo-ai/enhancements#16, "docs: add Human Identity RFC" (@craigmarker, open) -- proposes replacing `DummyAuth` for human-facing surfaces with OIDC plus a bespoke two-role model; see "Relationship to the Human Identity RFC" and Open questions above for how this proposal is scoped relative to it. michelangelo-ai/michelangelo#1504, "Add user identity to NavigationBar" (merged 2026-07-14) -- UI `UserRole` scaffolding already building on #16's model.
- Kubernetes: `SubjectAccessReview` (`authorization.k8s.io/v1`); `TokenReview` (`authentication.k8s.io/v1`); `TokenRequest` (`authentication.k8s.io/v1`, the `serviceaccounts/token` subresource behind the runner-token minting); `Role`/`RoleBinding`/`ClusterRole`/`ClusterRoleBinding`.
- k8s.io/apiserver contracts and building blocks adopted here: `pkg/authorization/authorizer` (`Authorizer`, `Attributes`, `AttributesRecord`); `pkg/authorization/authorizerfactory` (`DelegatingAuthorizerConfig`, `NewAlwaysAllowAuthorizer`); `pkg/authorization/union`; `pkg/authentication/authenticator` (`Token`, `Response`); `pkg/authentication/token/union` and the token cache; `pkg/authentication/user` (`user.Info`); `plugin/pkg/authenticator/token/oidc` and `plugin/pkg/authenticator/token/webhook` (candidate assemblies for the OIDC and TokenReview paths).
- Prior art for per-namespace execution identity: Argo Workflows (workflow ServiceAccount in the workflow's namespace), Kubeflow multi-user pipelines (per-profile `default-editor` SA), Tekton (`PipelineRun` ServiceAccount), Flyte (per-project/domain namespace identities).
- OIDC/OAuth2 standards: OpenID Connect Core 1.0; OpenID Connect Discovery 1.0 (the `/.well-known/openid-configuration` issuer-discovery mechanism this design uses); RFC 7517 (JSON Web Key); RFC 7519 (JSON Web Token); RFC 8414 (OAuth 2.0 Authorization Server Metadata, the OAuth-generic sibling of OIDC Discovery).
