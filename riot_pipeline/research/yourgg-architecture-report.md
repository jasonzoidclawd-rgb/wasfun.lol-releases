# YOUR.GG Technical Intelligence Report

Research date: 2026-06-30

Target: `https://your.gg/en/na/augments`

Scope: public, passive research only. No authentication bypass, private APIs, paid access, aggressive crawling, exploitation, credential discovery, or third-party use of the Riot API key was performed. The only Riot API key use was one official Riot endpoint request. That request returned 401, so key-backed Riot verification is incomplete.

## Executive Summary

YOUR.GG is a Next.js application with server-rendered pages, content-hashed immutable static chunks, styled-components server-side CSS, and a separate backend API namespace at `https://api.your.gg`. The `/en/na/augments` page is implemented as the Next route `/[region]/augments` and exposes `__N_SSP`, indicating server-side rendering/server-side props rather than a purely static page.

The augment page uses a public backend endpoint:

```text
GET https://api.your.gg/{region}/api/augment-guides/aram-mayhem
```

The client sends filters for `limit`, `cursor`, `tier`, `championKey`, `q`, and `mode`. Current public responses for both `na` and `kr` returned empty `items` with `patch: "16.13"`, but the page chunk reveals the intended card schema: `augmentId`, `name`, `iconUrl`, `tier`, `winRate`, `games`, `bestChampions`, `bestAugments`, and numeric `score` values mapped to OP/S/A/B/C/D/F labels.

Static metadata comes from public Riot ecosystem assets. Champion search responses include Data Dragon image URLs pinned to `16.13.1`. Arena augment metadata includes CommunityDragon image URLs under `raw.communitydragon.org/latest/game/assets/ux/cherry/augments/icons/...`. These static sources can explain names, icons, descriptions, versions, and patch assets. They cannot explain win rates, games played, best champions, best augment pairings, player reports, laning scores, team luck, or AI coaching.

The likely backend strategy is hybrid:

```text
Next.js SSR frontend
  -> public API backend at api.your.gg
    -> static metadata sync from Data Dragon and CommunityDragon
    -> Riot API ingestion for identity, rank, match, timeline, and live-game data
    -> batch/precomputed analytics tables by patch, region, mode, tier, champion, augment
    -> proprietary scoring/reporting/AI coaching layer
```

The strongest inference is that aggregate stats are precomputed. The public API returns no-store headers, but the data model is aggregate and patch-bucketed. Computing these metrics directly from live Riot API calls during page load would be slow, rate-limit intensive, and inconsistent with the frontend using pagination/cursors over materialized result rows.

## Methodology And Safety

| Method | Used | Notes |
| --- | --- | --- |
| Public HTML/header fetches | Yes | Target page, related route HEAD checks, manifest, robots. |
| Public JS bundle inspection | Yes | Only route names, non-sensitive endpoint patterns, and schema-bearing snippets were recorded. Raw app bundle was removed from `/tmp` after extraction. |
| Public endpoint fetches | Yes | Low-volume, browser-visible endpoints only. |
| Riot API key | Yes | One official Riot request to `na1.api.riotgames.com`; returned 401. |
| Playwright/DevTools HAR | No | Playwright was not installed locally. Controlled fetches and route manifest inspection were used instead. |
| Crawling match/multisearch pages | No | `robots.txt` disallows those URL families. |
| Authenticated or paid pages | No | Not accessed. |

## Diagram-Style Architecture Map

```text
Browser
  |
  | GET /en/na/augments
  v
your.gg nginx / Next.js
  |-- SSR HTML, __NEXT_DATA__, styled-components CSS
  |-- immutable _next/static JS/CSS chunks
  |-- public manifest, icons, splash images
  |
  +--> Analytics scripts
  |      Google Tag Manager / Analytics / Ads
  |      Microsoft Clarity
  |      Naver WCS
  |
  +--> Client runtime after hydration
         |
         | GET https://api.your.gg/{region}/api/augment-guides/aram-mayhem
         | GET https://api.your.gg/kr/api/search/champions?region=kr&lang={locale}
         | GET https://api.your.gg/{region}/api/metadata/arena/augments
         | GET https://api.your.gg/{region}/api/metadata/environment
         v
       api.your.gg backend
         |
         | static metadata joins
         +--> Riot Data Dragon CDN
         +--> CommunityDragon assets
         |
         | inferred server-side ingestion
         +--> Riot Account/Summoner/League/Match/Spectator APIs
         |
         | inferred storage/compute
         +--> match warehouse
         +--> patch/version metadata
         +--> aggregate analytics tables
         +--> proprietary score/report/AI layers
```

## Frontend Architecture

| Claim | Label | Evidence |
| --- | --- | --- |
| Framework is Next.js | Verified | Target HTML includes `x-powered-by: Next.js`, `__NEXT_DATA__`, `_next/static/...` chunks, and `_buildManifest.js`. |
| Route is `/[region]/augments` | Verified | `__NEXT_DATA__` has `"page":"/[region]/augments"` and query `{"region":"na"}`. |
| Page uses SSR/server-side props | Verified | Page chunk exports `__N_SSP`; HTML has large server-rendered `__NEXT_DATA__`; HTML cache header is private/no-store. |
| Locales are Next i18n locales | Verified | `__NEXT_DATA__` lists locales `default`, `ko`, `en`, `ja`, `es`, `vi`, `pt`; target locale `en`. |
| Regions are path-based | Verified | Build manifest route regex includes `kr|na|br|eune|euw|lan|las|oce|ru|tr|jp|sg|tw|vn`; client constants also include `me`. |
| Static assets are immutable | Verified | Next chunks return `cache-control: public, max-age=31536000, immutable`. |
| HTML is not browser-cacheable | Verified | Target HTML returns `cache-control: private, no-cache, no-store, max-age=0, must-revalidate`. |
| Styling uses styled-components | Verified | HTML includes `style data-styled` and `data-styled-version="5.3.3"`. |
| Client data fetching uses React Query style infinite query | Verified | Page chunk contains query key `aramMayhemAugments`, `getNextPageParam`, `staleTime:3e5`, `refetchOnWindowFocus:false`, and an `IntersectionObserver` for pagination. |
| Backend API host is `api.your.gg` | Verified | Public app chunk contains API base URL and generated client route names; public endpoint fetches succeeded against `https://api.your.gg`. |

## Routing And Related Pages

Verified routes from `_buildManifest.js` include:

| Route | Label | Notes |
| --- | --- | --- |
| `/[region]/augments` | Verified | Target page. |
| `/[region]/profile/[name]` | Verified | Profile pages likely use Riot identity/rank/match data. |
| `/[region]/match/[matchId]` | Verified | Exists, but robots disallows `/*/match/`; not crawled. |
| `/[region]/rift-chart` | Verified | Public route HEAD returned 200 and metadata endpoint returned chart metadata. |
| `/[region]/multisearch` | Verified | Exists, but robots disallows `/*/multisearch`; not crawled. |
| `/[region]/playreport/*` | Verified | Product/report pages and purchase flow. |
| `/rso/authentication/*` | Verified | Riot Sign On integration routes. |
| `/tools/admin/*` | Verified | Admin route names in public build manifest, not accessed. |

The related `/en/na/champions` route returned 404, while `/en/na/rift-chart` returned 200. The build manifest did not list `/[region]/champions`; champion data appears to be delivered through API endpoints and other pages rather than a standalone `/champions` route.

## Network Behavior And Endpoint Evidence

See `research/yourgg-network-inventory.md` and `research/yourgg-endpoints.csv` for the full request inventory. Key findings:

| Endpoint | Label | Evidence summary |
| --- | --- | --- |
| `GET https://api.your.gg/{region}/api/augment-guides/aram-mayhem` | Verified | Page chunk calls `getAramMayhemAugments`; current response has `hasMore`, `items`, `patch`, `statusCode`, `success`. |
| `GET https://api.your.gg/{region}/api/metadata/arena/augments` | Verified | Returns augment definitions with CommunityDragon image URLs. |
| `GET https://api.your.gg/{region}/api/search/champions` | Verified | Returns champion selector data with Data Dragon image URLs, lanes, and tier availability. |
| `GET https://api.your.gg/{region}/api/metadata/environment` | Verified | Returns public feature/status flag `riotApiErrorBannerEnabled`. |
| `GET https://api.your.gg/{region}/api/rift-chart/metadata` | Verified | Returns tiers, versions, tags, default tier/version, and updated timestamp. |
| `GET https://ddragon.leagueoflegends.com/api/versions.json` | Verified | Public Riot Data Dragon versions; latest observed `16.13.1`. |
| `HEAD https://raw.communitydragon.org/latest/.../warmuproutine_large.png` | Verified | Public CommunityDragon augment icon resolved as image/png. |

### Augment Page API Parameters

Verified in the public page chunk:

```text
limit        default 10
cursor       sent when paginating
tier         Prismatic | Gold | Silver, omitted for all
championKey  selected champion
q            search text
mode         Aram | Arena
```

The UI tabs map:

```json
{"mayhem":"Aram","arena":"Arena"}
```

The tier filter maps:

```json
{"all":null,"prismatic":"Prismatic","gold":"Gold","silver":"Silver"}
```

The numeric score label thresholds visible in the client:

```text
score >= 98 -> OP
score >= 92 -> S
score >= 75 -> A
score >= 50 -> B
score >= 25 -> C
score >= 10 -> D
else -> F
```

## Data Sources

| Claim | Label | Evidence |
| --- | --- | --- |
| YOUR.GG uses Data Dragon for champion imagery/metadata | Verified | Champion search response includes `https://ddragon.leagueoflegends.com/cdn/16.13.1/img/champion/Aatrox.png`; Data Dragon version endpoint returned `16.13.1`. |
| YOUR.GG uses CommunityDragon for Arena augment assets | Verified | Arena augment metadata response includes `https://raw.communitydragon.org/latest/game/assets/ux/cherry/augments/icons/...`; HEAD request to an exposed image returned 200 image/png. |
| YOUR.GG has internally normalized metadata beyond Data Dragon | Strong inference | Champion search returns lane lists and tier availability; Data Dragon champion JSON does not contain current lane/tier availability. |
| Aggregate augment analytics are internally precomputed | Strong inference | Client renders win rate, games, best champions, best augment pairings, and score; Riot APIs do not expose these aggregate tables directly; API response is patch-bucketed. |
| Profile/match/rank data comes from Riot APIs | Strong inference | Public route names include Riot ID search, summoners, profiles, matches, live-game, RSO routes. Riot APIs are the compliant source for these data types. |
| AI coaching/PlayReport uses proprietary scoring/modeling | Strong inference | Localization strings mention AI coaching, PLAYREPORT, laning, team luck, target metrics, and generated reports; Riot does not provide those outputs. |
| Third-party analytics/ad providers are used | Verified | Target HTML references Google analytics/tag scripts, Microsoft Clarity, and Naver WCS. |
| Public GitHub/npm references explain architecture | Unknown | Public web search for `your.gg`, `api.your.gg`, and `augment-guides` did not surface useful GitHub/npm implementation references. |

## Reconstructed Data Model

The detailed machine-readable schema is in `research/yourgg-inferred-schemas.json`. Summary:

| Entity | Label | Key fields |
| --- | --- | --- |
| `augmentGuideResponse` | Verified | `success`, `statusCode`, `response.hasMore`, `response.items`, `response.patch`, optional `response.nextCursor`. |
| `augmentGuideItem` | Strong inference | `augmentId`, `iconUrl`, `name`, `tier`, `shortDescription`, `description`, `winRate`, `games`, `bestChampions`, `bestAugments`. |
| `augmentBestChampion` | Strong inference | `championId`, `championKey`, `iconUrl`, `score`. |
| `augmentBestAugment` | Strong inference | `augmentId`, `iconUrl`, `name`, `score`. |
| `arenaAugmentMetadata` | Verified | `description`, `id`, `image`, `key`, `lanes`, `name`, `shortDescription`. |
| `championSearchItem` | Verified | `imageUrl`, `key`, `lanes`, `name`, `tiers`. |
| `riftChartMetadata` | Verified | `defaultTier`, `defaultVersion`, `tags`, `tiers`, `updated`, `versions`. |
| `playerOrSummoner` | Strong inference | `region`, Riot ID, `puuid`, summoner ids, profile icon, level, league entries. |
| `match` | Strong inference | `matchId`, `platformId`, `gameVersion`, queue, participants, teams, timeline. |
| `aggregateMetric` | Strong inference | patch, region, mode, tier bucket, champion, augment, games, wins/winRate, score, pairings. |

## Caching And Backend Strategy

| Layer | Label | Evidence | Interpretation |
| --- | --- | --- | --- |
| HTML | Verified | Target page private/no-store | SSR pages are regenerated per request or personalized/variant-sensitive. |
| Next static chunks | Verified | Immutable one-year cache headers | Standard content-hashed frontend deployment. |
| your.gg API JSON | Verified | API responses no-store/no-cache | Browser is told not to cache API responses; server may still use internal caches. |
| Data Dragon assets | Verified | CloudFront/S3 headers, CORS `*` | Public versioned CDN, safe to cache by patch/version. |
| CommunityDragon assets | Verified | Cloudflare, `cache-control: max-age=3600`, CORS `*` | Public asset cache with shorter TTL. |
| Augment analytics update cadence | Strong inference | Endpoint includes `patch: "16.13"`; query filters are aggregate buckets | Likely updated per patch with periodic refresh, not live per request. |
| Rift chart metadata cadence | Verified + inference | `updated: 1681960581109`, explicit versions | Some stat surfaces use materialized historical versions. |

Likely backend pattern:

1. Sync public metadata by game version/patch.
2. Collect and normalize Riot match/player/rank data server-side under API rate limits.
3. Materialize aggregate tables by patch, queue/mode, region, tier, champion, augment, and maybe confidence thresholds.
4. Serve fast paginated public APIs to the frontend.
5. Run separate personalized/reporting workflows for authenticated PlayReport and AI coaching products.

## Riot API vs your.gg Derived Analytics

Riot provides raw building blocks, not YOUR.GG's aggregate product.

| Data type | Riot provides directly | your.gg must compute or add |
| --- | --- | --- |
| Riot ID to PUUID | Account-V1 | Region/account linking, consent/account state. |
| Summoner profile | Summoner-V4 | Display model, caching, search UX. |
| Ranked tier/rank | League-V4 | Tier bucketing, historical tracking, filters. |
| Raw match detail | Match-V5 | Normalized participants, champion/lane inference, patch buckets. |
| Timeline events | Match-V5 timeline | Laning, snowball, objective, teamfight, positioning, contribution models. |
| Live active game | Spectator-V5 | Pre-game overlays, team composition scoring, live recommendations. |
| Champion/item/rune static data | Data Dragon | Lane/tier availability, patch-specific site metadata, localized UX. |
| Augment icons/descriptions | CommunityDragon/metadata | Augment performance, pairings, scores, search/filter ranking. |
| Win/pick rates | No | Aggregate from match corpus. |
| Best champions/augments | No | Aggregate and rank co-occurrence/outcome data. |
| OP/S/A score | No | Proprietary normalized scoring and thresholds. |
| AI coaching/report text | No | Proprietary model and product logic. |

## Risks And Limitations

| Risk/limitation | Label | Detail |
| --- | --- | --- |
| Riot API key check failed | Verified | Official status endpoint returned 401; no successful key-backed Riot API comparison was possible. |
| Augment analytics endpoint returned empty items | Verified | Current public `na` and `kr` responses show no rows. Item schema relies on client render code. |
| No DevTools HAR | Verified | Playwright not installed; controlled fetches were used instead. |
| Public bundles can expose sensitive-looking integration strings | Verified but not recorded | Bundle inspection was limited to route/schema extraction. Do not store raw bundles in the repo. |
| Paid/authenticated surfaces unknown | Unknown | My page, PlayReport, admin, payment, desktop app, and user-specific endpoints were not accessed. |
| Exact backend storage and scoring formulas unknown | Unknown | Public client shows thresholds and output fields, not backend SQL/model code. |
| Match/Multisearch crawling restricted | Verified | robots.txt disallows `/*/match/` and `/*/multisearch`; these were not crawled. |

## Suggestions For Building A Similar Compliant Analytics Pipeline

1. Keep all Riot API calls server-side. Never expose Riot keys in browser code.
2. Build an ingestion queue that understands Riot application and method limits, retries, and regional routing.
3. Normalize matches into append-only raw tables plus derived participant, timeline, champion, augment, and rank tables.
4. Pin every aggregate to a patch/version. Use Data Dragon version strings and a separate analytics patch bucket such as `16.13`.
5. Materialize public aggregate tables offline. Do not compute public win rates from live Riot calls.
6. Use Data Dragon for official static LoL assets and CommunityDragon only where appropriate for game data not present in Data Dragon.
7. Enforce sample-size thresholds and confidence penalties before showing win rate, pairings, or tier labels.
8. Keep public metadata APIs cacheable where possible, but serve personalized/account endpoints with no-store.
9. Respect robots.txt, Riot policies, user privacy, and deletion/consent requirements.
10. Document which metrics are direct Riot facts versus derived/proprietary analytics.

## Public References

| Reference | URL | Use |
| --- | --- | --- |
| Target page | `https://your.gg/en/na/augments` | Frontend and route evidence. |
| YOUR.GG robots | `https://your.gg/robots.txt` | Crawl boundaries. |
| YOUR.GG manifest | `https://your.gg/manifest.json` | PWA/static metadata. |
| YOUR.GG build manifest | `https://your.gg/_next/static/6etvsS_ZzfjyKeqIgnK-5/_buildManifest.js` | Route inventory. |
| Augments page chunk | `https://your.gg/_next/static/chunks/pages/%5Bregion%5D/augments-3ad2734e285f9307.js` | Client filters and schema. |
| YOUR.GG augment API | `https://api.your.gg/na/api/augment-guides/aram-mayhem?limit=2` | Endpoint shape and cache headers. |
| YOUR.GG arena augment metadata | `https://api.your.gg/na/api/metadata/arena/augments` | CommunityDragon metadata linkage. |
| YOUR.GG champion search | `https://api.your.gg/kr/api/search/champions?region=kr&lang=en` | Data Dragon linkage and champion selector schema. |
| Riot Developer docs | `https://developer.riotgames.com/docs/lol` | Official Riot API context. |
| Riot API reference | `https://developer.riotgames.com/apis` | Official endpoint families. |
| Data Dragon versions | `https://ddragon.leagueoflegends.com/api/versions.json` | Version source. |
| Data Dragon champion JSON | `https://ddragon.leagueoflegends.com/cdn/16.13.1/data/en_US/champion.json` | Champion static metadata. |
| CommunityDragon augment asset | `https://raw.communitydragon.org/latest/game/assets/ux/cherry/augments/icons/warmuproutine_large.png` | Augment asset source exposed by metadata. |

