---
name: appstorespy-market-discovery
description: >-
  Find and size a slice of the mobile app market with the AppstoreSpy REST API -
  filter-search the app database, pull chart rankings, find similar apps, get a
  summary of a filter set, and research Google Play keywords and LiveOps events.
api: AppstoreSpy API
base_url: https://api.appstorespy.com/v1
auth: API-KEY request header (Business plan required)
operations:
  - get_filter_search_ios_apps_query_post
  - get_filter_search_play_apps_query_post
  - get_summary_ios_apps_summary_post
  - get_summary_play_apps_summary_post
  - get_similar_ios_apps_similar_post
  - get_similar_play_apps_similar_post
  - get_ios_rankings_ios_rankings_get
  - get_play_rankings_play_rankings_get
  - dev_search_ios_developers_get
  - dev_search_play_developers_get
  - get_suggests_play_suggestions_get
  - get_events_play_liveops_get
  - create_search_jobs_search_post
  - get_search_jobs_search_get
generated: '2026-08-22'
method: generated
source: openapi/appstorespy-openapi.json + https://api.appstorespy.com/docs
---

# Size and explore a market segment

Base URL `https://api.appstorespy.com/v1`, headers `accept: application/json`
and `API-KEY: <your_token>`.

## 1. Summarise before you enumerate

Send your filter to `POST /ios/apps/summary` (`get_summary_ios_apps_summary_post`)
or `POST /play/apps/summary` (`get_summary_play_apps_summary_post`) FIRST. The
summary tells you how large the result set is for a fraction of the credits it
would cost to page through it. Only then decide whether to enumerate.

## 2. Filter-search the database

`POST /ios/apps/query` (`get_filter_search_ios_apps_query_post`) or
`POST /play/apps/query` (`get_filter_search_play_apps_query_post`).

The request body is `IosAppSearchBody` / `PlayAppSearchBody`, built from the
`SearchFilterIos` / `SearchFilterPlay` shapes with `ValueRange` and `DateRange`
constructs (installs between X and Y, released between two dates, and so on).
Categories come from the `IosCategory` / `PlayCategory` enums - use the enum
values from the contract, not free text.

Control the response with:

- `fields=` - return only the metrics you will actually use.
- `sort=` / `sort=-` - ascending or descending, from `IosSortEnum` / `PlaySortEnum`.
- `page` and `limit` - offset pagination; `total_count` comes back in the
  envelope so you can tell when to stop.

## 3. Read the charts

`GET /ios/rankings` (`get_ios_rankings_ios_rankings_get`) and
`GET /play/rankings` (`get_play_rankings_play_rankings_get`) take `country`,
`category`, `collection`, `rank_start`/`rank_end` and a `date_start`/`date_end`
window, and return ranking rows (`date`, `app`, `country`, `category`,
`collection`, `rank`). This is the right surface for "who is winning this
category over time", not the app-detail endpoint.

## 4. Expand around a seed

`POST /ios/apps/similar` (`get_similar_ios_apps_similar_post`) or
`POST /play/apps/similar` (`get_similar_play_apps_similar_post`) takes a set of
app IDs and a `SimilarEnum` mode and returns the neighbourhood. Use it to widen
a niche you found by filter, then re-summarise.

## 5. Find the operators

`GET /ios/developers` (`dev_search_ios_developers_get`) and
`GET /play/developers` (`dev_search_play_developers_get`) search the developer
side of the same database - the right move when the question is "who builds in
this space" rather than "which apps exist".

## 6. Keyword and LiveOps research (Google Play)

- `GET /play/suggestions` (`get_suggests_play_suggestions_get`) - keyword
  suggestions for a `term`, scoped by `country`/`lang`, returning `Suggest` with
  `results[]` of `Cluster`.
- `GET /play/liveops` (`get_events_play_liveops_get`) - LiveOps `Event` records
  for an app (`type`, `title`, `text`, `image`, `video`, `date_from`, `date_to`).

Both return **`202 Accepted`** when the term or app has not been crawled yet.
That is a queue receipt, not a failure: wait and re-request the same URL.

## 7. Long-running store searches

For a store search you want done asynchronously, `POST /jobs/search`
(`create_search_jobs_search_post`) with a `SearchCreate` body (`store`, `term`,
`country`, `lang`, `limit` up to 250) and retrieve it later with
`GET /jobs/search?search_id=...` (`get_search_jobs_search_get`), which returns
`SearchResult` with `results[]` of `Cluster`.

There is **no cancel operation** for a job. Once created it runs and the credits
are spent - create one deliberately.

## Rules that apply to every step

- Inspect `errors[]` inside a `200` list envelope; partial failures live there.
- `422` carries `detail[].loc` / `detail[].msg` / `detail[].type` - fix the
  parameter rather than retrying.
- `429` has no `Retry-After`; back off on your own schedule.
- Every call spends API credits and there is no idempotency key, so a blind
  retry is charged twice. Budget against the plan quota (Business: 100,000
  credits/month per the pricing page) and note the per-endpoint credit cost is
  not published.
