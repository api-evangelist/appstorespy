---
name: appstorespy-app-intelligence-lookup
description: >-
  Look up a single mobile app on the App Store or Google Play and assemble its
  full intelligence picture - metadata, developer, reviews, chart rankings,
  download/revenue estimates and daily installs - using the AppstoreSpy REST API.
api: AppstoreSpy API
base_url: https://api.appstorespy.com/v1
auth: API-KEY request header (Business plan required)
operations:
  - get_app_details_ios_apps__id__get
  - get_app_details_play_apps__id__get
  - get_developer_ios_developers__id__get
  - get_developer_play_developers__id__get
  - get_reviews_ios_apps__id__reviews_get
  - get_reviews_play_apps__id__reviews_get
  - get_apps_estimates_ios_estimates_get
  - get_apps_estimates_play_estimates_get
  - get_apps_installs_daily_play_apps__id__installs_daily_get
  - get_ios_rankings_ios_rankings_get
  - get_play_rankings_play_rankings_get
  - get_play_app_countries_ios_info_countries_get
  - get_play_app_countries_play_info_countries_get
generated: '2026-08-22'
method: generated
source: openapi/appstorespy-openapi.json + https://api.appstorespy.com/docs
---

# Look up one app and build its intelligence picture

Every request goes to `https://api.appstorespy.com/v1` with two headers:

```
accept: application/json
API-KEY: <your_token>
```

Get the token from https://appstorespy.com/account. API access requires the
Business plan; without a key the edge answers `401 {"message":"Not
authenticated","code":401}`.

## 1. Pick the store, because the identifier space differs

- App Store: `id` is the numeric Apple app ID (e.g. `284882215`) - use the
  `/ios/...` paths.
- Google Play: `id` is the package name (e.g. `com.snapchat.android`) - use the
  `/play/...` paths.

If you only have a name, resolve it first with `GET /ios/apps?q=...`
(`get_app_search_ios_apps_get`) or `GET /play/apps?q=...`
(`get_app_search_play_apps_get`).

## 2. Check which countries the app actually has data for

`GET /ios/info/countries` (`get_play_app_countries_ios_info_countries_get`) or
`GET /play/info/countries` (`get_play_app_countries_play_info_countries_get`).

Requesting a country the app is not available in returns `204 No Content` - an
empty body, not an error. Check availability before spending credits on a
country-scoped read.

## 3. Fetch the app record

`GET /ios/apps/{id}` (`get_app_details_ios_apps__id__get`) or
`GET /play/apps/{id}` (`get_app_details_play_apps__id__get`).

Use `fields=` to fetch only what you need - `fields=name,developer_name,ipd,downloads`
- and omit it only when you genuinely want all 50+ properties. Fewer fields is
cheaper to parse and easier to reason about.

Handle these:

- `206 Partial Content` (App Store only) - a partial record; treat missing
  fields as unknown, not as zero.
- `202 Accepted` - the app was not in the database and has been queued for
  crawling. Do not retry in a tight loop; come back later.

## 4. Follow the developer edge

The app record carries `developer_id` and `previous_developer_id` (set when an
app has been transferred). Read the owner with `GET /ios/developers/{id}`
(`get_developer_ios_developers__id__get`) or `GET /play/developers/{id}`
(`get_developer_play_developers__id__get`).

For the developer's whole book, use
`GET /ios/developers/{id}/estimates` (`get_apps_estimates_ios_developers__id__estimates_get`)
or `GET /play/developers/{id}/estimates` (`get_apps_estimates_play_developers__id__estimates_get`).

## 5. Add the time series

- Monthly downloads and revenue: `GET /ios/estimates`
  (`get_apps_estimates_ios_estimates_get`) or `GET /play/estimates`
  (`get_apps_estimates_play_estimates_get`) - `AppsEstimates` rows keyed by app
  `id` and `month`.
- Daily installs (Google Play only): `GET /play/apps/{id}/installs_daily`
  (`get_apps_installs_daily_play_apps__id__installs_daily_get`) - one row per
  `date` with `ipd` and `installs`.
- Chart position: `GET /ios/rankings` (`get_ios_rankings_ios_rankings_get`) or
  `GET /play/rankings` (`get_play_rankings_play_rankings_get`), dimensioned by
  `date`, `country`, `category`, `collection`.

## 6. Add reviews

`GET /ios/apps/{id}/reviews` (`get_reviews_ios_apps__id__reviews_get`) or
`GET /play/apps/{id}/reviews` (`get_reviews_play_apps__id__reviews_get`).
Reviews are country-specific - pass `country` and page with `page`/`limit`.

## Rules that apply to every step

- **Read `errors[]` on success.** List responses are
  `{ data: [...], errors: [...], total_count: n }`. A `200` can still report
  per-item failures in `errors[]` (`location`, `message`, `value`).
- **Validation failures are `422`** with a FastAPI-shaped body:
  `detail[].loc` points at the offending parameter, `detail[].type` names the
  rule. Fix the parameter; do not retry unchanged.
- **`429` means back off.** No `Retry-After` or `RateLimit-*` header is
  published or returned, so use your own exponential backoff.
- **Every call spends credits** and there is no idempotency key. A retried
  request is charged again - do not retry on timeouts without checking whether
  the first call succeeded.
- **Never trigger a recrawl in a loop.** `GET /ios/apps/{id}/recrawl` and
  `GET /play/apps/{id}/recrawl` queue work and cannot be cancelled.
