# YOUR.GG Network Inventory

Research date: 2026-06-30

Scope: public, passive inspection of `https://your.gg/en/na/augments`, related public routes, public JavaScript bundles, metadata files, and official Riot/public asset endpoints. No authentication bypass, private API access, fuzzing, high-volume crawling, or credential discovery was performed. Playwright was not installed in this local workspace, so the inventory is based on controlled fetches of browser-visible documents, route manifests, page chunks, and low-volume public API requests.

## Robots And Crawl Rules

Verified: `https://your.gg/robots.txt` returned HTTP 200 as `text/plain`.

Relevant rules:

| Rule | Evidence | Impact |
| --- | --- | --- |
| `User-agent: * Allow: /` | robots.txt | Target page and static assets are allowed for normal public access. |
| `Disallow: /*/match/` | robots.txt | Match detail pages should not be crawled. This research did not crawl match pages. |
| `Disallow: /*/multisearch` | robots.txt | Multisearch pages should not be crawled. This research did not crawl multisearch pages. |
| Commercial SEO crawlers blocked | robots.txt lists several named bots | Confirms the site explicitly tries to reduce unwanted SSR egress from automated crawlers. |

## Target Page Load Inventory

Target: `https://your.gg/en/na/augments`

| Category | Public request | Method | Observed status | Notes |
| --- | --- | --- | --- | --- |
| HTML document | `https://your.gg/en/na/augments` | GET/HEAD | 200 | Server-rendered Next.js document. `x-powered-by: Next.js`. `cache-control: private, no-cache, no-store, max-age=0, must-revalidate`. |
| Manifest | `https://your.gg/manifest.json` | GET | 200 | PWA manifest with app name, icons, standalone display. Public static file with `cache-control: public, max-age=0`. |
| Robots | `https://your.gg/robots.txt` | GET | 200 | Public static file with `cache-control: public, max-age=0`. |
| Sitemap | `https://your.gg/sitemap.xml` | GET | 404 | Served by Next.js error page, not a sitemap. |
| CSS | `https://your.gg/_next/static/css/caedf0901b5c49b7.css` | GET | Referenced | Immutable Next static asset. |
| Core JS | `https://your.gg/_next/static/chunks/webpack-2ff8442418ffbb76.js` | GET | Referenced | Next webpack runtime chunk. |
| Core JS | `https://your.gg/_next/static/chunks/framework-16775d0b28321a07.js` | GET | Referenced | React/Next framework chunk. |
| Core JS | `https://your.gg/_next/static/chunks/main-c560f3ee5ebf66f7.js` | GET | 200 when fetched | Immutable static chunk. |
| App JS | `https://your.gg/_next/static/chunks/pages/_app-e7c448694e032c43.js` | GET | 200 when fetched | Large application chunk containing generated API client route names and public client configuration. Credential-like values from this public bundle were not recorded. |
| Shared JS | `https://your.gg/_next/static/chunks/6637-7195c5ae232402ac.js` | GET | 200 when fetched | Shared utility/styling chunk. |
| Shared JS | `https://your.gg/_next/static/chunks/3828-b1e0713ceffa3ee3.js` | GET | 200 when fetched | Shared chunk. |
| Page JS | `https://your.gg/_next/static/chunks/pages/%5Bregion%5D/augments-3ad2734e285f9307.js` | GET | 200 when fetched | Augments page logic and UI schema. |
| Build manifest | `https://your.gg/_next/static/6etvsS_ZzfjyKeqIgnK-5/_buildManifest.js` | GET | 200 | Route manifest for many Next pages. |
| SSG manifest | `https://your.gg/_next/static/6etvsS_ZzfjyKeqIgnK-5/_ssgManifest.js` | GET | Referenced | Next static generation manifest. |
| Middleware manifest | `https://your.gg/_next/static/6etvsS_ZzfjyKeqIgnK-5/_middlewareManifest.js` | GET | Referenced | Next middleware manifest. |
| Fonts | `https://cdn.jsdelivr.net/.../SUIT.css`, `https://cdn.jsdelivr.net/.../pretendard...css`, Google Fonts | GET | Referenced | Public font CSS. |
| Analytics | Google Tag Manager, Google Analytics/Ads, Microsoft Clarity, Naver WCS | GET/script | Referenced | Public browser analytics scripts. Public identifiers are omitted here because they are not needed for architecture reconstruction. |

## Static Asset Header Pattern

Verified chunk headers for `_app`, `main`, `6637`, `3828`, and the augments page chunk:

| Header | Observed value pattern |
| --- | --- |
| `server` | `nginx/1.20.2` |
| `content-type` | `application/javascript; charset=UTF-8` |
| `cache-control` | `public, max-age=31536000, immutable` |
| `last-modified` | `Fri, 26 Jun 2026 05:31:37 GMT` |
| `etag` | Weak ETags such as `W/"cbe9-19f026982a8"` |

Verified implication: Next static chunks are content-hashed and browser/cache friendly. HTML and API responses are not cached by browsers.

## API/Data Endpoints Observed

### Augment analytics endpoint

Verified from page chunk and public fetch:

```text
GET https://api.your.gg/{region}/api/augment-guides/aram-mayhem
```

Client parameters visible in the page chunk:

| Parameter | Source | Notes |
| --- | --- | --- |
| `limit` | Verified in page chunk | Default client value is `10`; low-volume test used `2`. |
| `cursor` | Verified in page chunk | Used for infinite loading when `hasMore` is true. |
| `tier` | Verified in page chunk | Values mapped by client: `Prismatic`, `Gold`, `Silver`; omitted for all tiers. |
| `championKey` | Verified in page chunk | Selected champion key, for example `Aatrox`. |
| `q` | Verified in page chunk | Search text. |
| `mode` | Verified in page chunk | Client maps UI tabs to `Aram` and `Arena`. |

Observed response for both `na` and `kr` with `limit=2`:

```json
{"response":{"hasMore":false,"items":[],"patch":"16.13"},"statusCode":200,"success":true}
```

Observed headers:

| Header | Observed pattern |
| --- | --- |
| `content-type` | `application/json` |
| `cache-control` | `no-cache, no-store, max-age=0, must-revalidate` |
| `pragma` | `no-cache` |
| `expires` | `0` |
| `x-frame-options` | `SAMEORIGIN` |
| `vary` | `Origin`, `Access-Control-Request-Method`, `Access-Control-Request-Headers` |

Inference: response shape is live/backend API JSON, but the data is probably precomputed batch analytics because it returns aggregate patch/tier/champion/augment metrics rather than a live Riot API lookup.

### Arena augment metadata endpoint

Verified:

```text
GET https://api.your.gg/{region}/api/metadata/arena/augments
```

Response shape example:

```json
{
  "data": [
    {
      "description": "Gain the Warmup Routine Summoner Spell...",
      "id": 0,
      "image": "https://raw.communitydragon.org/latest/game/assets/ux/cherry/augments/icons/warmuproutine_large.png",
      "key": "93",
      "lanes": [],
      "name": "Warmup Routine",
      "shortDescription": ""
    }
  ]
}
```

Evidence: image URLs point to CommunityDragon. Response headers are no-store API headers, not CDN immutable headers.

### Champion search endpoint

Verified from page chunk and public fetch:

```text
GET https://api.your.gg/kr/api/search/champions?region=kr&lang=en
```

Response shape example:

```json
[
  {
    "imageUrl": "https://ddragon.leagueoflegends.com/cdn/16.13.1/img/champion/Aatrox.png",
    "key": "Aatrox",
    "lanes": ["Top"],
    "name": "Aatrox",
    "tiers": ["Unranked", "Iron", "Bronze", "Silver", "Gold", "Platinum", "Emerald", "Diamond", "Master", "Grandmaster", "Challenger"]
  }
]
```

Evidence: the page chunk calls `searchChampionsUsingGet({ region: "kr", lang: router.locale })` for the augment champion selector. This is a notable implementation detail: the selector fetch uses `kr` rather than the target page region.

### Environment endpoint

Verified:

```text
GET https://api.your.gg/na/api/metadata/environment
```

Response:

```json
{"riotApiErrorBannerEnabled":false}
```

Inference: public feature/status flags are exposed through metadata APIs.

### Rift chart metadata endpoint

Verified:

```text
GET https://api.your.gg/na/api/rift-chart/metadata
```

Response excerpt:

```json
{
  "defaultTier": "PlatinumPlus",
  "defaultVersion": "13.01",
  "tiers": ["Challenger", "GrandMaster", "GrandMasterPlus", "Master", "MasterPlus", "Diamond", "DiamondPlus", "Platinum", "PlatinumPlus", "Gold", "Silver", "Bronze", "Iron"],
  "updated": 1681960581109,
  "versions": ["13.01", "12.23"]
}
```

Inference: at least some chart/stat surfaces are backed by versioned, precomputed metadata distinct from live Riot APIs.

## Public Third-Party Asset Sources

| Source | Verified public URL | Evidence |
| --- | --- | --- |
| Riot Data Dragon versions | `https://ddragon.leagueoflegends.com/api/versions.json` | Returned latest `16.13.1`, CloudFront/S3 headers, `Access-Control-Allow-Origin: *`. |
| Riot Data Dragon champion metadata | `https://ddragon.leagueoflegends.com/cdn/16.13.1/data/en_US/champion.json` | Returned champion JSON with `version`, `id`, `key`, `name`, `image`, `stats`. |
| CommunityDragon augment image | `https://raw.communitydragon.org/latest/game/assets/ux/cherry/augments/icons/warmuproutine_large.png` | HEAD returned 200 image/png, Cloudflare, CORS `*`, `cache-control: max-age=3600`. |

## Related Route Signals

Verified from `_buildManifest.js`:

| Route pattern | Evidence | Inference |
| --- | --- | --- |
| `/[region]/augments` | Build manifest and target route | Augment guide surface. |
| `/[region]/profile/[name]` | Build manifest | Profile pages likely SSR and use profile/match/league APIs. |
| `/[region]/match/[matchId]` | Build manifest and robots disallow | Match detail pages exist but should not be crawled. |
| `/[region]/rift-chart` | Build manifest and public route HEAD 200 | Precomputed champion/chart analytics surface. |
| `/[region]/multisearch` | Build manifest and robots disallow | Multisearch exists but should not be crawled. |
| `/[region]/my/playreport`, `/[region]/playreport/*` | Build manifest | Paid or account-linked report surfaces. |
| `/rso/authentication/*` | Build manifest | Riot Sign On integration is likely used for account connection. |

## Limitations

| Limitation | Effect |
| --- | --- |
| Playwright was not installed | No full DevTools HAR was captured. Request inventory is reconstructed from normal document, bundle, manifest, and controlled endpoint fetches. |
| Riot API key returned 401 on the single official test request | Key-backed Riot verification is incomplete. Riot comparison uses public Riot docs and public asset endpoints. |
| Augment analytics endpoint returned empty `items` for `na` and `kr` | Item schema is inferred from client rendering code, not from a populated live payload. |
| No private authenticated surfaces accessed | Paid/account-only PlayReport, user dashboard, desktop app, and internal admin workflows remain unknown. |

