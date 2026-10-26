# Meta Ads dashboard — project handoff

Prepared: 27 October 2026 (Asia/Jakarta).

## Instructions for the next session

Read this file before taking action. It contains the user's objective, completed research, limits, and proposed next steps. It is self-contained and can be copied into a new project directory.

The user has requested feasibility research and this handoff. No dashboard implementation has been started or explicitly commissioned yet. Continue from the user's next instruction; do not treat this document as authorization to change live ads, budgets, permissions, or publish a service.

Do not repeat broad feasibility research. Recheck time-sensitive details when implementing, then prioritize a real account integration proof once the user asks to build.

## User's objective

The user wants a Meta Ads Manager dashboard clone with a familiar Campaigns → Ad sets → Ads hierarchy and a fast, reliable interface. Their requested capabilities include, but are not limited to:

- Create and manage campaigns.
- Create and manage ad sets.
- Create and manage ads.
- Bulk upload ads and creative assets.
- Turn campaigns, ad sets, and ads on and off, including bulk operations.
- Change campaign settings and budgets.
- Change ad set settings and budgets.

Their motivation is avoiding the lag, bugs, and glitches of Meta's original dashboard.

No framework, hosting, database, visual design, account count, campaign type, or implementation timeline has been selected. The user has not specified whether the product is for their own accounts only or for unrelated clients.

## Conversation and completed work

1. User asked whether the dashboard was feasible.
2. Assistant researched the official API and answered yes, with limits imposed by Meta's backend.
3. User explicitly requested more research.
4. Assistant examined current Meta developer documentation and Meta's official Python Business SDK, including creation/update parameters, budgets, batches, uploads, statuses, reporting, webhooks, authorization, and version changes.
5. Assistant concluded the research supports designing the product and starting an integration prototype. Account compatibility remains untested.
6. User asked to move work to another directory and requested a handoff file.

No Meta app was created, no account connected, no access token inspected, no ad created, no campaign activated, and no budget changed. No application code, database, deployment, or tests exist for this dashboard.

A previous response linked `docs/meta-ads-dashboard-research.md`, and its creation was verified in that turn, but that file was absent when preparing this handoff. Do not rely on it. The substantive research is preserved below.

The old working directory was `/home/fast9/Secondary/Ecom/English Lesson/Code`. It contains unrelated English-lesson/landing-page files and an `auto_weekly_recap` directory. Do not assume they are part of the new dashboard or copy them into it.

## Feasibility conclusion

The core workflows are feasible through Meta's official Marketing API. Build a custom application backed by that API.

A fast UI is achievable through local caching, drafts, background processing, and explicit sync state. Meta still controls request acceptance, processing, ad review, delivery, and reporting freshness. Do not promise a bug-free Meta backend, instant publication, finalized real-time spend, or universal parity with Ads Manager.

A focused first version is realistic. Full coverage of every campaign type, creative format, Advantage+ feature, and account-specific option is a larger scope requiring individual verification.

Source: [Marketing API overview](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview).

## Access: important correction to the first feasibility answer

There are separate layers of access:

| Layer | Meaning | Implication |
| --- | --- | --- |
| Permission access level | Standard or Advanced access to permissions such as `ads_management` and `ads_read` | Meta documents Standard permission access for own accounts and Advanced permission access for other people's accounts. |
| Marketing API Access Tier | Limited or Full API access/capacity | Limited is the default development tier. Meta explicitly describes it as unsuitable for production apps serving live advertisers. Full requires App Review. |
| Asset assignment | Rights to particular ad accounts, Pages, pixels/datasets, Instagram identities, and other assets | A token's scopes do not establish rights to every asset. |

The initial statement that Standard Access is enough for own accounts was incomplete as a production plan. Permission access and API tier must be assessed separately, even for an internal tool.

The documented Full-tier qualification includes at least 500 successful Marketing API calls in the preceding 15 days and an error rate below 15% over the last 500 calls. This is not a promise of approval or its timing. Exact requirements must be checked in the app dashboard.

Business verification depends on app/permission usage. Meta requires verified business ownership for Advanced permission access and apps allowing other businesses to access their data, while documenting exceptions for app-role-only use. Do not assume either that verification is always required for initial testing or that own-account use removes all production review requirements.

For internal server automation, evaluate a system user with assigned assets. For unrelated customer onboarding, evaluate the appropriate authorization flow and Advanced permissions. Never ask the user to paste secrets into a public file or commit them.

Sources: [Authorization](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization), [Business verification](https://developers.facebook.com/docs/apps/business-verification), [System users](https://developers.facebook.com/docs/marketing-api/system-users).

## API objects and supported operations

| Operation | Mechanism | Notes |
| --- | --- | --- |
| Create campaign | `POST /act_{account_id}/campaigns` | Validate objective, special category, bidding, and budget mode together. |
| Create ad set | `POST /act_{account_id}/adsets` | Targeting, promoted object, optimization, placements, and attribution must form an accepted combination. |
| Create creative | `POST /act_{account_id}/adcreatives` | Creative is a separate object from the ad. |
| Create ad | `POST /act_{account_id}/ads` | References the ad set and creative. |
| Update objects | `POST /{object_id}` | Supported fields and transitions depend on object type and current configuration. |
| Pause/activate | Update `status` | Show configured state and effective delivery state separately. |
| Duplicate | Copy operations or source fields | Revalidate old settings rather than blindly replaying them. |
| Preview | Ad/creative preview endpoints | Verify the formats and placements actually used. |

Official SDK methods confirm broad support. An SDK parameter or enum being present does not prove it is accepted for every account, version, campaign type, or post-launch transition. Some enums retain legacy values.

Creative content generally cannot be edited in place after creation. Meta documents limited metadata/status edits; image, text, and video changes should use a replacement creative specification or creative ID on the ad. Review and engagement behavior must be verified for each supported format.

Sources: [Ad account SDK](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/adobjects/adaccount.py), [Campaign SDK](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/adobjects/campaign.py), [Ad set SDK](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/adobjects/adset.py), [Ad SDK](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/adobjects/ad.py), [Creative SDK](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/adobjects/adcreative.py), [Updating creatives through ads](https://developers.facebook.com/documentation/ads-commerce/marketing-api/asyncrequests).

## Budgets and editable settings

Support campaign budgets and independent ad set budgets as distinct modes. Under Advantage campaign budget, ad set minimum spend targets/caps are different controls from independent budgets.

Meta documents restrictions for Advantage campaign budget, including optimization-goal editing after ads run and restrictions on changing bid strategy/disabling campaign budget when there are more than 70 ad sets. Do not present every field as universally editable.

Amounts use the account currency's Meta offset. USD has offset 100; IDR has offset 1. Therefore Rp100,000 maps to `100000` for an IDR account. Do not multiply every currency by 100. The user's location does not establish the account's actual currency or timezone; read both from the account.

Recommended UI: display budget mode, period, currency, timezone, resulting allocations, and pending/confirmed amounts. Validate daily/lifetime settings and applicable minimums. Treat changing budget mode as an explicit guided operation.

Sources: [Advantage campaign budget](https://developers.facebook.com/documentation/ads-commerce/marketing-api/bidding/guides/advantage-campaign-budget), [Currency offsets](https://developers.facebook.com/docs/marketing-api/currencies/).

## Rate limits that shape the product

As researched on 27 October 2026, Meta documents:

- Ad set budget edits: four per hour; exceeding this blocks further budget changes for an hour (`613/1487632`).
- Same-object editing: default one edit per object every 30 seconds, per endpoint, with possible endpoint-specific differences (`613/4841018`).
- Mutation bursts: 100 requests/second per app/account combination for campaign/ad set/ad creates and edits. This is an additional ceiling, not a guaranteed sustained throughput.
- Account scoring: Limited/development maximum 60, Full maximum 9,000; reads generally cost 1, writes 3, with decay over 300 seconds.
- Additional account/use-case, application, Insights, and other limits apply.

Pace jobs using actual usage headers and error subcodes. Do not handle every `613` identically. Do not assume more parallelism improves speed.

Recommended behavior: local drafts, explicit combined saves, serialization per object, merging compatible unsent edits, visible delays, and adaptive retries. Prioritize pause requests over routine uploads where possible, while showing any enforced delay. An immediate UI response must not imply Meta already accepted a pause.

Source: [Marketing API rate limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting).

## Bulk uploads and reliable recovery

Dependency chain: create required campaign/ad set → upload media → wait for readiness → create creative → create ad.

Meta supports:

- Ordinary batches of up to 50 requests; recommends 10 or fewer ad creations per batch.
- A separate asynchronous batch interface documenting up to 1,000 requests, dependencies, and progress inspection.
- Per-operation success/failure; a whole batch is not an atomic transaction.

Ordinary batch calls count individually toward quotas. The Graph batch guide states that a batch cannot include multiple ad sets under the same campaign. Start with small batches and serialize same-campaign ad set creation; verify actual behavior before optimizing.

Video upload completion is separate from encoding readiness. Meta's official uploader implements start/transfer/finish and a separate readiness check.

Recommended upload workflow:

1. Files plus a table/CSV containing metadata and target objects.
2. Validation and previews per row, including identity and destination.
3. Durable job and step records that survive refreshes and worker restarts.
4. Persist returned media, creative, and ad IDs after each successful step.
5. Create the initial objects paused and expose activation as a distinct action.
6. Retry only the necessary steps with known outcomes.

Use application operation IDs and asset checksums to limit accidental duplication. These are not assumed Meta-enforced idempotency keys. A create timeout can mean the object was created but its response was lost. Reconcile before resubmission; if identity remains uncertain, flag it for resolution instead of automatically creating another ad. Readback may itself be delayed, so an immediate absence is not conclusive.

Sources: [Asynchronous and batch requests](https://developers.facebook.com/documentation/ads-commerce/marketing-api/asyncrequests), [Graph batches](https://developers.facebook.com/docs/graph-api/batch-requests/), [Official video uploader](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/video_uploader.py).

## Synchronization, status, and conflicts

Meta now documents ad-account webhooks for effective status, object subscriptions, processing completion, and issue states. Object subscriptions can signal creation and selected metadata changes.

Notifications trigger targeted API reads; they are not complete replacement object records. Setup requires an app subscription plus connecting each ad account. The callback verifies a challenge and validates `X-Hub-Signature-256` over the raw body.

Recommended model:

- Serve tables from a local database with a visible last-sync time.
- Keep desired state, observed Meta state, and operation progress separate.
- Read back submitted changes and refresh on webhook notifications.
- Periodically reconcile missed/unsupported events.
- Detect conflicting edits made through Ads Manager before overwriting fields.
- Show configured on/off state alongside effective delivery state. An active ad may still be blocked by a parent, review, or another delivery condition.

Sources: [Ads webhooks](https://developers.facebook.com/documentation/ads-commerce/marketing-api/ads-webhooks/ads-webhooks-overview), [Object subscriptions](https://developers.facebook.com/documentation/ads-commerce/marketing-api/ads-webhooks/subscriptions), [Setup](https://developers.facebook.com/documentation/ads-commerce/marketing-api/ads-webhooks/setup/get-started), [Effective status](https://developers.facebook.com/documentation/ads-commerce/marketing-api/ads-webhooks/effective-status).

## Reporting

Meta documents approximately 15-minute Insights refreshes, account-timezone reporting, revisions after ads finish, and asynchronous jobs for large queries. Do not promise finalized per-second spend/conversions.

Keep reporting refreshes separate from configuration/status sync. Match timezone, date range, attribution context, and metric definitions when comparing against Ads Manager. Re-fetch a recent rolling window for revisions. Large historical queries should not block table navigation.

Source: [Insights limits and best practices](https://developers.facebook.com/documentation/ads-commerce/marketing-api/insights/best-practices).

## API versions and current changes

These are dated research findings, not permanent facts. Revalidate before implementation.

The dedicated Marketing API changelog listed v26.0 as latest, v25.0 available until 23 February 2027, and v24.0 until 10 November 2026. Marketing API and general Graph API retirement schedules differ. Some secondary sources had conflicting dates; use Meta's dedicated Marketing changelog.

The official Python SDK configuration retrieved during research listed API v26.0 / SDK v26.0.2. No SDK has been selected for this project.

The v26.0 Marketing changelog includes poll creative removal, restrictions on `web_only` app-link treatment for Web and App campaigns, explicit Advantage+ audience flags for certain constrained regulated-category targeting, and removed delivery-estimate fields. Several changes apply across versions on 27 October 2026. It also removes Instagram Explore Feed and describes Messenger Stories removal. Normalize copied templates.

The Graph v26.0 changelog removes root `GET /?ids=...` reads and legacy ETag/304 behavior, with cross-version changes dated 27 October 2026. Some older Marketing examples still recommend these patterns. Prefer per-object reads and supported batching; verify the adapter against the chosen version.

Sources: [Marketing changelog](https://developers.facebook.com/documentation/ads-commerce/marketing-api/marketing-api-changelog), [Marketing v26](https://developers.facebook.com/documentation/ads-commerce/marketing-api/marketing-api-changelog/version26.0), [Graph v26](https://developers.facebook.com/docs/graph-api/changelog/version26.0/), [SDK configuration](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/apiconfig.py).

## Proposed architecture — recommendations, not selected technology

| Component | Responsibility |
| --- | --- |
| Table UI | Hierarchy, filters, selection, drafts, forms, bulk operations, progress |
| Backend API | Sessions, account authorization, validation, Meta calls, secrets, audit |
| Relational database | Cached objects, drafts, job steps, remote IDs, conflicts, history |
| Worker and durable queue | Uploads, paced writes, recovery, readback, reporting |
| Object storage | Source assets, checksums, previews, retention |
| Webhook receiver and scheduler | Targeted refresh plus periodic reconciliation |

Keep tokens and app secrets on the server. Meta supports `appsecret_proof` for server calls. Redact credentials from logs, including pagination URLs. Authorize every account-specific operation.

Preserve useful API error fields (`code`, `error_subcode`, user-facing message, trace ID). Distinguish validation failure, missing permission, expired/revoked token, throttling, processing failure, and unknown outcome.

Sources: [Secure requests](https://developers.facebook.com/docs/graph-api/guides/secure-requests), [Error handling](https://developers.facebook.com/docs/graph-api/guides/error-handling).

## Recommended first implementation milestone

Once the user asks to build, narrow the initial campaign/creative types and prove the integration before investing in complete UI parity:

1. Inventory app permissions, tier, verification, token identity, and asset assignments.
2. Read one account's currency, timezone, existing hierarchy, and required assets.
3. Create one paused campaign → ad set → creative → ad chain for a real supported use case.
4. Verify budget edits in the correct currency/mode and status readback, controlling activation explicitly.
5. Process a small image/video batch with an invalid row and verify independent results.
6. Verify restart recovery and ambiguous-outcome handling without blind repeated creates.
7. Verify webhooks, external edits, and periodic reconciliation.
8. Compare a bounded performance report against Ads Manager with matching reporting context.

No timeline or throughput estimate has been established. Actual app/account access and behavior are the remaining integration uncertainties.

## Decisions still needed

- Own accounts only or unrelated client accounts?
- First campaign types: website sales/leads, instant forms, WhatsApp/Messenger, catalog, or app promotion?
- Accounts/object counts and expected upload batch sizes?
- Required images, videos, carousels, existing posts, or other creative formats?
- Existing Meta app, API tier, verified business, identities, pixels/datasets, and assignments?
- Number of operators and permissions to activate ads/change budgets?
- Preferred stack, hosting, authentication, and deployment arrangement?

Do not ask all of these again as a long questionnaire if useful context can be inspected or the user already supplies a direction. No answers have been recorded yet.

## Research evidence and tooling notes

Public source downloads remain at `/tmp/meta-ads-research` in the current environment: plain-text/HTML documentation and selected official Python SDK files under `facebook_business/`. This temporary cache may disappear and is not required to use this handoff. Do not treat it as application code.

The web tool frequently returned 429/access errors for Meta documentation. Direct public HTTPS downloads worked using Python urllib, `?locale=en_US`, and BeautifulSoup text extraction. Some older URLs redirect to `/documentation/ads-commerce/...`. Some reference pages show older version selectors even while current guides and SDK use v26; distinguish documentation evidence from live verification.

The shell environment had an `SSLKEYLOGFILE` pointing outside the writable workspace, causing initial HTTPS downloads to fail. Fetches used `env -u SSLKEYLOGFILE` to avoid writing TLS session keys. Respect the new directory's permissions and instructions; do not assume old approvals or writable roots transfer.

No task-specific AGENTS.md was found in the old workspace during research. Inspect the new workspace and its applicable instructions before editing.
