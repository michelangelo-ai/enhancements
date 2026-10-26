# RFC-20261026-public-documentation-hosting: Agent-friendly documentation delivery

- **Status:** Draft
- **Author(s):** @austingreco
- **Created:** 2026-10-26
- **Internal ERD:** Not linked. Maintainers should confirm applicability before advancing this draft; any required organizational approvals remain internal.

---

## Problem statement

Our public documentation at [michelangelo-ai.org](https://michelangelo-ai.org) is built with Docusaurus and hosted on GitHub Pages. This is a simple, low-cost setup that works well for people reading the website.

AI coding assistants can read those pages too, but receive browser-oriented HTML containing navigation and presentation elements. We want to offer clean Markdown at the same public URLs without changing the website experience or documentation framework. GitHub Pages alone does not provide the request-handling control needed to select between these representations.

Read-only checks on October 26, 2026 confirmed that both `/` and `/docs/` returned HTML for `Accept: text/markdown`. A nonexistent path correctly returned HTTP 404, but also returned HTML. These responses varied on `Accept-Encoding`, not `Accept`. No new automated scan was triggered.

## Motivation

People and agents should receive consistent guidance from the same documentation. A clean text representation can make relevant instructions easier to extract without presentation markup. Maintaining the existing URLs avoids a separate documentation address that agents must discover and users must keep synchronized.

This is a delivery improvement, not a claim that agents cannot use the current website. Success should be measured by correct content and HTTP behavior, not a scanner score. Moving hosts alone will not produce useful Markdown: the export work is required whichever platform we choose.

**Recommendation:** use Amazon S3 and CloudFront, with a small CloudFront Function for format selection. This is a familiar, inexpensive static-hosting pattern that does not require an application server or container.

## Goals

- Preserve the existing website experience, public URLs, redirects, and documentation authoring workflow.
- Serve HTML or clean Markdown at the same URL according to the request, including the homepage and relevant generated index pages.
- Generate both representations from the same source revision and prevent them from contaminating each other's cached responses.
- Preserve real HTTP 404s and provide useful Markdown errors when requested.
- Keep deployment, ownership, operating cost, and rollback straightforward.

## Non-goals

- Redesigning the website or replacing Docusaurus.
- Adding a product API, MCP server, or CLI to improve a scanner score.
- Rendering documentation dynamically or converting HTML to Markdown on every request.
- Moving the domain registrar or DNS provider.
- Changing Michelangelo platform APIs, Kubernetes resources, or runtime infrastructure.
- Authorizing provisioning or a production cutover merely by opening this draft.

## High-level architecture

```text
GitHub Actions
  -> existing Docusaurus build + Markdown exports + public-route manifest
  -> private S3 bucket

Readers and agents
  -> CloudFront Function selects HTML or Markdown
  -> CloudFront cache
  -> S3 on a cache miss
```

Keep the existing dependency installation, changelog synchronization, and documentation build behavior. Add a provider-neutral export package containing HTML, clean Markdown, and a public-route manifest. Use actual route information rather than guessing URLs from source filenames. Normalize links and images, preserve code and warnings, and produce meaningful Markdown for the React homepage and generated pages.

A viewer-request CloudFront Function runs before cache lookup. It selects the representation and rewrites internally to a distinct object path while preserving the public URL. Static images, JavaScript, and CSS do not need format negotiation. The implementation must preserve canonical redirects and avoid redirecting visitors to internal artifact paths.

Keep S3 private using CloudFront Origin Access Control. Use a standard ACM certificate and retain the existing UltraDNS provider. Confirm the root-domain alias configuration before cutover; Route 53 is not required.

GitHub Actions should deploy through a narrowly scoped AWS role using OIDC. Upload complete artifacts before activating a release, retain bounded rollback versions, and configure budget alerts. Alerts are not hard spending caps. Account ownership, deployment access, and billing responsibility must be agreed before implementation.

## APIs and CRDs

No Michelangelo platform API or CRD changes are proposed. The public documentation HTTP behavior changes as follows:

| Request | Proposed behavior |
|---|---|
| Existing page with `Accept: text/html` | HTML with the correct content type |
| Existing page with `Accept: text/markdown` | Markdown at the same URL with the correct content type |
| Missing `Accept`, `*/*`, or equally preferred supported formats | HTML by default |
| Mixed preferences | Honor media-range specificity and quality values, including explicit exclusions |
| Existing page with neither supported format acceptable | HTTP 406 |
| Unknown page with a supported format preference | HTTP 404 with a useful error in that format |

Negotiated responses must include `Vary: Accept`, preserving other applicable values, and representation-specific validators. Choose the representation before evaluating conditional requests. The default HTML rule must not override an explicit exclusion of HTML.

Validation must cover both cache-fill orders, repeated requests through CloudFront, conditional GET and HEAD, redirects, the homepage, generated pages, and unknown paths. Do not turn missing objects into successful SPA responses. Private S3 missing-object behavior and CloudFront Function code-size limits must be addressed in the implementation rather than assuming every origin error is a missing page.

## Alternatives considered

All four options retain Docusaurus. Compare the simplest suitable implementation on each platform, not matching product names component by component.

| Option | Benefits | Main tradeoff | Illustrative monthly cost |
|---|---|---|---|
| **1. AWS: S3 + CloudFront (recommended)** | Familiar static hosting with the required request control; no application server or container | We maintain a small routing function and deployment workflow; requires a project-owned AWS account | **Under $1** with available allowances; **about $5** otherwise |
| **2. Vercel** | Integrated deployments, previews, and rollback with flexible request handling | Platform subscription; additional deploying seats and usage can increase costs | **From $20** on Pro |
| **3. Google: Firebase Hosting + a small request handler** | Includes CDN, HTTPS, previews, and rollback; no separately configured load balancer | A small Cloud Run service or Cloud Function selects HTML or Markdown | **About $6-7** with available allowances |
| **4. Keep GitHub Pages** | No migration or incremental hosting cost; separate Markdown files could still be published | Same-URL format selection requires another serving layer | **$0 incremental** |

These are USD planning estimates based on pricing checked October 23, 2026, not measured traffic or guaranteed bills. The model assumes 100,000 pageviews/month, six requests and 0.5 MB per pageview (600,000 requests and 50 GB), North American delivery, 95% request and byte cache-hit rates, approximately 1 GB retained, and 30 deployments uploading 1,000 objects each where object-operation charges apply. Vercel assumes one deploying seat and light backend/build use within its included credit. Firebase assumes a small cached backend that fits available compute allowances; exact backend/network billing needs validation.

CloudFront's recurring allowance covers 1 TB of delivery, 10 million HTTP/HTTPS requests, and 2 million Function invocations monthly. These allowances are not dedicated to this site. Actual traffic, other account usage, retention, region, logs, and team seats can change the bill. Existing domain/DNS and shared CI costs, engineering time, taxes, and optional add-ons are excluded.

AWS is recommended for its familiar, lightweight operating model, not because the alternatives are expensive. Firebase is a credible low-cost alternative; Vercel's packaged workflow may justify its subscription. Staying on GitHub Pages is reasonable if separate Markdown URLs are sufficient. The migration buys delivery control, not savings over the current host.

## Open questions

- [ ] Which project-owned AWS account, billing owner, and operational owner should host the site? Available allowances and required organizational approvals are not confirmed.
- [ ] Who administers the existing DNS zone, and what root-domain alias configuration should be used?
- [ ] Does this public documentation-only proposal require a linked internal ERD under the contribution policy? Confirm before advancing the draft; do not treat this RFC as an internal approval.
- [ ] Which live version milestone should track implementation and cutover? Neither repository listed an open milestone on October 26, 2026.
- [ ] What are current traffic and transfer volumes, and what alert thresholds and retention limits should we adopt?
- [ ] Does the community agree that same-URL negotiation justifies migration, rather than retaining GitHub Pages with separate Markdown files?

## Rollout strategy

This RFC remains **Draft** pending maintainer review, internal-ERD applicability, and milestone mapping. Follow the repository's shepherd and final-comment-period process; do not begin production migration before acceptance.

The process requires rollout to map to upcoming live GitHub version milestones. The [core repository milestones](https://github.com/michelangelo-ai/michelangelo/milestones) and [enhancements milestones](https://github.com/michelangelo-ai/enhancements/milestones) had no open milestones when checked on October 26, 2026. The shepherd must identify or establish the target milestone and update the mapping below before acceptance. This draft does not invent a release version or a cutover date.

| Phase | Release mapping | Exit criteria |
|---|---|---|
| Review and ownership | Draft; target milestone unresolved | Agreement on provider, ownership, approvals, success criteria, and live milestone |
| Build and preview | First implementation milestone selected by the shepherd | Synchronized exports and routing pass content, cache, HTTP, and browser checks on a temporary hostname |
| Production cutover | Same milestone or its agreed successor, after preview approval | Certificate and DNS prerequisites confirmed; canonical URLs and both formats work in production |
| Stabilization and cleanup | Follow-up work linked to the agreed milestone | Costs and errors reviewed; rollback window completed before retiring the old deployment |

Implementation issues and PRs should link back to the accepted RFC and carry the selected release milestone. The website deploys independently of platform runtime code; milestone tracking must not imply a user-facing platform upgrade is required.

Keep GitHub Pages and the previous DNS configuration available during the rollback window. A failed cutover should restore the previous routing and site rather than trying to repair production in place. Validate both successful deployment and rollback before retiring the old setup.

## References

- [Michelangelo RFC process](../README.md)
- [Current documentation deployment workflow](https://github.com/michelangelo-ai/michelangelo/blob/main/.github/workflows/deploy-docs.yml)
- [CloudFront Function execution](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/function-code-choose-purpose.html)
- [CloudFront cache keys](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/understanding-the-cache-key.html)
- [CloudFront origin access control](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [CloudFront custom domains](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html)
- [CloudFront pricing](https://aws.amazon.com/cloudfront/pricing/pay-as-you-go/) and [S3 pricing](https://aws.amazon.com/s3/pricing/)
- [Vercel Pro](https://vercel.com/docs/plans/pro-plan) and [Routing Middleware](https://vercel.com/docs/routing-middleware)
- [Firebase Hosting caching](https://firebase.google.com/docs/hosting/manage-cache), [Cloud Run integration](https://firebase.google.com/docs/hosting/cloud-run), and [pricing](https://firebase.google.com/pricing)
- [HTTP content negotiation and validators](https://www.rfc-editor.org/rfc/rfc9110.html)
