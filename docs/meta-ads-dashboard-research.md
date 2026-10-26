# Custom Meta Ads dashboard: implementation research

Researched 27 October 2026. Scope: an internal dashboard to create and manage campaigns, ad sets, and ads, including bulk uploads, status changes, settings, and budgets.

## Assessment

The requested core workflows are feasible through Meta's official Marketing API. A responsive interface is achievable. Actual publication, processing, delivery, reporting freshness, and accepted edits remain dependent on Meta. This is enough research to design a focused product and begin an integration proof; it is not evidence that a particular account or every Ads Manager feature is supported.

This investigation used public Meta documentation and Meta's Business SDK source. No account was connected, no token was inspected, and no ad or budget was changed. Findings below distinguish documented behavior from recommended implementation choices. [Meta Marketing API overview](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview)

## 1. Access: an important correction to the initial answer

Two independent concepts must be checked:

| Concept | Purpose | Implication |
| --- | --- | --- |
| Permission access levels | Standard or Advanced access to permissions such as `ads_management` and `ads_read` | Meta documents Standard permission access for your own ad accounts; other people's accounts need Advanced permission access. |
| Marketing API Access Tier | Limited or Full access to the Marketing API | Limited is the default development tier. Meta explicitly describes it as unsuitable for production apps serving live advertisers. Full requires App Review. |

Therefore, “own accounts need only Standard Access” is incomplete as a production plan. Standard permission access and production API capacity are separate decisions. Meta lists at least 500 successful Marketing API calls in the preceding 15 days and an error rate below 15% over the last 500 calls for upgrading to Full access. Account asset assignments are another independent requirement. [Authorization](https://developers.facebook.com/documentation/ads-commerce/marketing-api/get-started/authorization)

Business verification must also be checked. Meta requires verified business ownership for apps requesting Advanced permission access and apps allowing other businesses to access their data, while documenting exceptions for app-role-only use. The exact review requirements for this app must be confirmed in its dashboard; approval timing cannot be inferred from these pages. [Business verification](https://developers.facebook.com/docs/apps/business-verification)

For an internal service, evaluate a system user assigned the necessary ad accounts, Pages, and other assets. For onboarding unrelated customers, evaluate the appropriate user authorization flow and Advanced permissions. A token alone does not establish access to every asset. [System users](https://developers.facebook.com/docs/marketing-api/system-users)

## 2. Supported workflows and boundaries

| Workflow | API mechanism | Boundary to reflect in the product |
| --- | --- | --- |
| Create campaigns | `POST /act_{account_id}/campaigns` | Validate objective, special category, budget mode, and bidding together. |
| Create ad sets | `POST /act_{account_id}/adsets` | Targeting, promoted object, optimization, placements, and attribution must form an accepted combination. |
| Create ads | `POST /act_{account_id}/ads` | Requires the appropriate ad set and creative. |
| Edit campaign settings | `POST /{campaign_id}` | Exposed parameters do not guarantee every transition is permitted on an existing campaign. |
| Edit ad set settings | `POST /{adset_id}` | Availability depends on the campaign type, existing configuration, and delivery history. |
| Pause or activate | Update `status` on the relevant object | Configured status and effective delivery status must be shown separately. |
| Duplicate | Campaign/ad set/ad copy operations and source fields | Revalidate inherited settings; a copied configuration may be obsolete. |
| Preview | Ad or creative preview endpoints | Validate actual formats and placements used by the account. |

The official SDK exposes these creation and update methods. Its enums also contain legacy values, so the SDK must not become the sole source of valid UI choices. [Ad account methods](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/adobjects/adaccount.py), [campaign methods](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/adobjects/campaign.py), [ad set methods](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/adobjects/adset.py), [ad methods](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/adobjects/ad.py)

Creative editing needs special treatment. Meta documents that a creative's content cannot be edited after creation, apart from limited metadata/status changes. To change an ad's creative, provide a replacement creative specification or a new creative ID. Design copy/image/video editing around that operation and verify review and engagement behavior for each format. [Asynchronous and batch requests — updating ads](https://developers.facebook.com/documentation/ads-commerce/marketing-api/asyncrequests)

The first version should explicitly support selected campaign and creative types. Website conversions, instant forms, messaging, catalog ads, app ads, and partnership ads have different dependencies. Full Ads Manager parity is a separate, substantially larger scope.

## 3. Budgets require distinct modes

Campaign budgets and independent ad set budgets are both supported. Under Advantage campaign budget, Meta distributes the campaign budget across ad sets; ad set spend limits or minimum targets are different controls from independent budgets. Meta also documents restrictions, including optimization-goal editing after ads run in this setup, and restrictions on changing bid strategy or disabling campaign budget when a campaign has more than 70 ad sets. These must not be represented as universally editable fields. [Advantage campaign budget](https://developers.facebook.com/documentation/ads-commerce/marketing-api/bidding/guides/advantage-campaign-budget)

Amounts use Meta's account-currency offset. USD has offset 100; IDR has offset 1. For an IDR account, Rp100,000 maps to `100000`, rather than multiplying by 100 as with dollar cents. Read currency from the account; use integer amounts and the documented currency table. [Currency codes](https://developers.facebook.com/docs/marketing-api/currencies/)

Recommendation: show budget mode, currency, period, and account timezone explicitly. Validate daily versus lifetime settings and account-specific minimums before submission. Treat changing budget mode as a guided operation with the resulting campaign and ad set allocations visible for review.

## 4. Rate limits affect the interaction design

Meta currently documents:

- Ad set budget changes: four per hour; exceeding this blocks further budget changes for an hour.
- Same-object editing: a default of one edit every 30 seconds, per object and endpoint; individual endpoint limits can differ.
- Campaign/ad set/ad mutation bursts: 100 requests per second per app/account combination. This is an additional ceiling, not sustainable throughput permission.
- Development account score: maximum 60; Full-access score: 9,000. Reads generally score 1 and writes 3, with decay over 300 seconds.
- Separate account/use-case, application, and Insights limits also apply.

Use error subcodes and returned usage headers to identify the relevant limiter. In particular, budget throttling uses `613/1487632`; same-object concurrency throttling uses `613/4841018`. Do not treat every `613` as equivalent. [Marketing API rate limiting](https://developers.facebook.com/documentation/ads-commerce/marketing-api/overview/rate-limiting)

Recommended behavior:

- Save local drafts immediately; submit explicit, combined changes rather than one API call per keystroke.
- Serialize writes to each object. Combine compatible unsent changes, preserving the latest intended value.
- Keep pause requests ahead of routine uploads where possible, while displaying any enforced delay.
- Show pending and confirmed values separately. A responsive toggle must not imply that Meta has already paused delivery.
- Use adaptive pacing and delayed retries. More simultaneous requests can make reliability worse.

## 5. Bulk uploading is a resumable workflow

The dependency chain is: upload media → wait for processing when necessary → create creative → create ad under the intended ad set. Campaign and ad set creation precede these steps when new parents are needed.

Meta supports ordinary batches of up to 50 requests, and recommends 10 or fewer ad creations per batch. Its separate asynchronous batch interface documents up to 1,000 requests with dependencies and progress inspection. These are different mechanisms, not interchangeable batch-size settings. [Asynchronous and batch requests](https://developers.facebook.com/documentation/ads-commerce/marketing-api/asyncrequests)

Ordinary batch operations count individually toward API limits, can partially fail, and the Graph guide states that a batch cannot include multiple ad sets under the same campaign. Begin with small batches and serialized same-campaign ad set creation; verify actual behavior before optimizing. [Graph batch requests](https://developers.facebook.com/docs/graph-api/batch-requests/)

Video upload completion does not establish encoding readiness. Meta's official uploader implements start/transfer/finish stages and a separate readiness check. The dashboard should show uploading and processing as separate states. [Official video uploader](https://github.com/facebook/facebook-python-business-sdk/blob/main/facebook_business/video_uploader.py)

Recommended initial bulk workflow:

1. Select files and import ad metadata through a table or CSV.
2. Validate required fields and display the target account, campaign, ad set, identity, and destination for every row.
3. Preview and save the batch as a durable job.
4. Process rows in the background, persisting every returned media, creative, and ad ID.
5. Create initial objects paused and present results for activation.
6. Resume unfinished work after a refresh or worker restart; retry only failed steps whose outcomes are known.

Use application operation IDs and asset checksums to prevent accidental repeated submission. Do not assume that these are Meta-enforced idempotency keys. A timeout after a create request can leave its outcome unknown. Reconcile before retrying; if identity cannot be established confidently, mark the row for resolution. Retrying an entire partially successful job can create duplicate ads.

## 6. Synchronization can use webhooks

Meta now documents ad-account webhooks for effective status, object subscriptions, processing completion, and issue states. Object subscriptions can signal creation and selected metadata changes. A notification should trigger a targeted API read; it is not a complete replacement object. [Ads webhooks](https://developers.facebook.com/documentation/ads-commerce/marketing-api/ads-webhooks/ads-webhooks-overview), [object subscriptions](https://developers.facebook.com/documentation/ads-commerce/marketing-api/ads-webhooks/subscriptions)

Setup requires both an app-level subscription and connecting individual ad accounts. The documented callback verifies a challenge and validates `X-Hub-Signature-256` over the raw request body. [Webhook setup](https://developers.facebook.com/documentation/ads-commerce/marketing-api/ads-webhooks/setup/get-started)

Recommended synchronization model:

- Serve tables from a local database, with a visible last-sync time.
- Read back changes after submission and reconcile webhook-triggered updates.
- Periodically reconcile as a fallback for missed or unsupported events.
- Detect external edits made through Ads Manager before overwriting conflicting fields.
- Keep desired state, last observed Meta state, and operation progress separate.

Show configured on/off state alongside effective delivery status. An active ad can still be blocked by its parent campaign, review, or another delivery condition. [Effective-status webhooks](https://developers.facebook.com/documentation/ads-commerce/marketing-api/ads-webhooks/effective-status)

## 7. Reporting needs its own refresh cycle

Meta documents approximately 15-minute Insights refreshes, account-timezone reporting, asynchronous reporting jobs for larger queries, and revisions after ads finish. It also documents attribution behavior intended to align API reporting with Ads Manager. A fast dashboard cannot manufacture second-by-second finalized spend or conversion data. [Insights limits and best practices](https://developers.facebook.com/documentation/ads-commerce/marketing-api/insights/best-practices)

Recommendation: keep performance refreshes separate from settings/status synchronization, show metric timestamps, and compare reports using matching timezone, date range, attribution context, and metric definitions. Re-fetch a recent rolling window to capture revisions. Do not make a large lifetime reporting query block campaign-table navigation.

## 8. Version changes are relevant today

Meta's dedicated Marketing API changelog currently lists v26.0 as the latest version, v25.0 available until 23 February 2027, and v24.0 until 10 November 2026. Use a centrally pinned supported version and schedule compatibility maintenance. Do not substitute the generic Graph API version lifespan for Marketing API dates. [Marketing API changelog](https://developers.facebook.com/documentation/ads-commerce/marketing-api/marketing-api-changelog)

The v26.0 changelog includes poll creative removal, restrictions on `web_only` app-link treatment for Web and App campaigns, explicit Advantage+ audience flags for certain constrained regulated-category targeting, and removed delivery-estimate fields. Several apply across versions on 27 October 2026. It also removes Instagram Explore Feed and describes removal of Messenger Stories. Old templates need normalization rather than blind replay. [Marketing API v26.0 changes](https://developers.facebook.com/documentation/ads-commerce/marketing-api/marketing-api-changelog/version26.0)

The Graph v26.0 changelog also removes root `GET /?ids=...` reads and legacy ETag/304 behavior, with cross-version changes dated 27 October 2026. Some older Marketing examples still recommend these patterns. Prefer per-object reads and supported batching, and validate the adapter against the selected version rather than copying legacy examples. [Graph API v26.0 changes](https://developers.facebook.com/docs/graph-api/changelog/version26.0/)

## 9. Proposed application structure

These are engineering recommendations, not Meta requirements:

| Component | Responsibility |
| --- | --- |
| Table interface | Campaign/ad set/ad navigation, filters, bulk selection, drafts, editable settings, upload progress |
| Backend API | Sessions, account authorization, validation, Meta calls, secrets, audit records |
| Relational database | Cached objects, draft versions, durable job steps, returned Meta IDs, conflicts, history |
| Background worker and queue | Media processing, paced mutations, retries, readback verification, reporting |
| Object storage | Source uploads, checksums, temporary previews, controlled retention |
| Webhook receiver and scheduler | Targeted refreshes plus periodic reconciliation |

Keep tokens and app secrets on the server. Meta supports `appsecret_proof` for server calls. Redact credentials from logs, including API pagination URLs, and authorize every account-specific operation. [Secure Graph requests](https://developers.facebook.com/docs/graph-api/guides/secure-requests)

Store useful API error details and show the actionable message. Differentiate validation failure, lost permission, token failure, throttling, processing failure, and an unknown request outcome. Meta documents user-facing error fields and trace IDs. [Error handling](https://developers.facebook.com/docs/graph-api/guides/error-handling)

## 10. First integration proof before building the full interface

1. Inventory the app's permission levels, API tier, business verification, token identity, and asset assignments.
2. Read one account's currency, timezone, campaigns, ad sets, ads, and relevant identities/assets.
3. Validate one real campaign type and create a paused campaign/ad set/creative/ad chain.
4. Verify budget edits in the account's currency and budget mode; verify status changes and readback without activating spend unless explicitly intended.
5. Run a small image/video batch with one deliberately invalid row. Confirm that successful rows survive and only unresolved rows need attention.
6. Exercise worker interruption and ambiguous outcomes without blindly repeating creations.
7. Verify external edits, webhook setup, and periodic reconciliation.
8. Compare a bounded report against Ads Manager with matching reporting context.

The integration should pass these checks before claiming production reliability. No performance estimate or approval timeline is justified until the actual account and tier have been observed.

## Remaining account-specific decisions

- Own business accounts only, or accounts belonging to unrelated clients?
- Which campaign types come first: website sales/leads, instant forms, WhatsApp/Messenger, catalog, or app promotion?
- How many accounts, existing objects, and uploads per batch?
- Which creative formats and identities must be supported?
- What app, access tier, business verification, and assigned assets already exist?
- How many operators can edit simultaneously, and who can activate ads or change budgets?

These decisions narrow implementation scope. The public research supports proceeding to that integration proof; live account compatibility and end-to-end behavior remain untested.
