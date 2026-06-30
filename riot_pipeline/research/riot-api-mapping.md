# Riot API Mapping For YOUR.GG-Like Analytics

Research date: 2026-06-30

This file maps official Riot API capabilities and public Riot asset sources against the public evidence observed on `your.gg`. The single allowed Riot API verification request was made to an official Riot endpoint using the environment-provided Riot token. The token value was not printed, logged, committed, or stored.

## Riot API Key Verification

Status: Verified incomplete.

Request:

```text
GET https://na1.api.riotgames.com/lol/status/v4/platform-data
Header: Riot authentication header read from the environment-provided token
```

Observed result:

```json
{"status":{"message":"Forbidden","status_code":401}}
```

Interpretation: an environment-provided Riot token was present but was missing required authorization, expired, or otherwise invalid for this request at the time of testing. No further Riot API requests were made with the token.

## Official Riot Sources And What They Provide

| Source | Public/official URL | What it provides | What it does not provide |
| --- | --- | --- | --- |
| Riot Developer Portal | `https://developer.riotgames.com/docs/lol` and `https://developer.riotgames.com/apis` | Official LoL API documentation, routing regions, endpoint families, auth model, policy context | Does not provide your.gg proprietary schemas or aggregates |
| Account-V1 | Developer Portal API docs | PUUID lookup by Riot ID, active shard by game | No match statistics, win rates, or augments aggregates |
| Summoner-V4 | Developer Portal API docs | Encrypted summoner identifiers, profile icon, summoner level by platform | No Riot ID tag lookup by itself; no aggregate analytics |
| Match-V5 | Developer Portal API docs | Match IDs by PUUID, full match detail, match timeline by regional routing cluster | No precomputed champion win rates, augment scores, synergies, counters, or tier buckets |
| League-V4 | Developer Portal API docs | Ranked entries, queue tier/rank/LP/wins/losses | No per-match details or champion/augment statistics |
| Spectator-V5 | Developer Portal API docs | Current game info for an active summoner | No historical aggregation; no win/pick rates |
| Data Dragon | `https://ddragon.leagueoflegends.com/api/versions.json` and versioned CDN paths | Static versioned champion/item/rune/spell/assets metadata | No match history, rank, or aggregate statistics |
| CommunityDragon | `https://raw.communitydragon.org/latest/...` | Community-maintained extracted game data/assets, including Arena augment icons used by your.gg metadata | Not an official Riot API; no player match history or aggregate analytics |

## Riot API vs your.gg Derived Analytics

| Visible your.gg feature/data | Directly available from Riot? | Likely source path | Required computation |
| --- | --- | --- | --- |
| Champion icons and names | Yes, from Data Dragon | Data Dragon champion JSON and image CDN | Minimal mapping/version normalization |
| Arena augment names/descriptions/icons | Not through standard Riot API; visible via CommunityDragon and your.gg metadata | CommunityDragon asset paths exposed by `api.your.gg` metadata | Normalize augment IDs/keys, localization, and patch availability |
| Player profile by Riot ID | Partly | Account-V1 for PUUID, Summoner-V4 for summoner data, League-V4 for rank | Merge identity, region routing, cache updates |
| Match history | Yes, raw match IDs/details from Match-V5 | PUUID to Match-V5 list/detail/timeline | Store, normalize participants, queues, patches, timelines |
| Live game | Yes, from Spectator-V5 for active games | Summoner or PUUID mapping to spectator endpoint | Short-lived cache and UI derivation |
| Win rate, pick rate, games played by champion/tier/patch | No | Derived from collected Match-V5 rows | Batch aggregation by patch, region, queue/mode, champion, tier |
| Augment win rate and games played | No | Derived from Arena/ARAM Mayhem match data if augments are encoded in match payloads or timelines | Extract augment selections, join to participants, compute wins/games |
| Best champions for an augment | No | Derived from aggregate match outcomes | Group by augment and champion; rank by score/win rate/sample confidence |
| Best augment pairings | No | Derived from co-occurrence in collected matches | Pair/co-occurrence aggregation and score normalization |
| OP/S/A/B/C/D/F score labels | No | Proprietary scoring layer | Normalize win rate, sample size, popularity, confidence, patch recency, possibly champion/tier context |
| "Rating", "Laning", "Team Luck", "Carry", "Snowball" style metrics visible in localization strings | No | Derived from Match-V5 timeline and internal models | Timeline feature engineering and proprietary model/heuristics |
| AI PlayReport/coaching | No | Internal product layer using Riot match data plus models | Prompt/model pipeline, subscription state, report generation |

Conclusion: Riot APIs can supply raw identity, rank, live-game, and match/timeline records. The visible analytics that make `your.gg` useful - win rates, scores, best pairings, team luck, laning, carry, matchup insights, and AI report content - must be computed independently from collected data and product-specific models.

## Rate Limit Implications

Because the test key returned 401, live rate-limit headers were not available from a successful call. A compliant implementation should still assume:

| Design concern | Practical implication |
| --- | --- |
| Per-application and per-method limits | Centralize Riot API calls behind a quota-aware scheduler; do not let frontend traffic call Riot directly. |
| Match-V5 fanout | One player lookup can expand to many match IDs and then many match details. Avoid recursive crawling and deduplicate match IDs globally. |
| Patch analytics scale | Use batch ingestion and aggregation jobs; do not compute public aggregate pages from live Riot calls. |
| Regional routing | Map platform regions such as NA1/KR/EUW1 to regional clusters such as AMERICAS/ASIA/EUROPE for Match-V5. |
| Cacheability | Static metadata can be cached by patch; player/profile and live-game data need shorter TTLs; aggregate analytics should be materialized. |
| Compliance | Use Riot API only for official endpoints, honor Riot policies, do not expose API keys to the browser, and store only permitted data. |

## Suggested Compliant Pipeline

1. Identity resolution: Riot ID -> Account-V1 PUUID -> platform-specific Summoner-V4 and League-V4 records.
2. Match ingestion: fetch recent Match-V5 IDs for opted-in or searched players within quota; deduplicate match IDs before detail fetches.
3. Normalization: flatten matches into participants, teams, objectives, items, runes, augments if present, and timelines.
4. Metadata sync: periodically sync Data Dragon versions and CommunityDragon augment metadata; pin every derived row to a patch/version.
5. Aggregation: compute materialized tables by patch, region, queue/mode, tier bucket, champion, augment, and sample-size thresholds.
6. Scoring: apply documented scoring formulas with confidence penalties for low sample size and recency weighting.
7. Serving: expose your own cached API for frontend pages. Use no-store for personalized endpoints; cache static chunks and patch metadata aggressively.
8. Governance: keep API keys server-side only, implement rate-limit backoff, monitor policy changes, and provide deletion/consent workflows where required.

## Mapping To Observed your.gg Endpoints

| your.gg endpoint/surface | Riot source likely involved | Evidence label | Evidence |
| --- | --- | --- | --- |
| `/api/search/champions` | Data Dragon champion metadata plus internal lane/tier availability | Verified + strong inference | Response includes Data Dragon image URLs with version `16.13.1`; lanes/tiers are not raw Data Dragon fields and are likely internal/stat-derived. |
| `/api/metadata/arena/augments` | CommunityDragon augment assets plus internal metadata normalization | Verified | Response image fields point to `raw.communitydragon.org/latest/game/assets/ux/cherry/augments/icons/...`. |
| `/api/augment-guides/aram-mayhem` | Internal aggregate match analytics | Strong inference | Endpoint response includes `patch`; page chunk renders aggregate `winRate`, `games`, best champions and best augment pairings. Riot does not expose these directly. |
| `/api/profile`, `/api/summoners`, `/api/search/by-riot-id/*` | Account-V1, Summoner-V4, League-V4, Match-V5 | Strong inference | Generated public client route names include Riot ID search, summoner, profile, and profile version endpoints. |
| `/api/live-game/*` | Spectator-V5 plus internal live tracking | Strong inference | Generated public client route names include live game, widgets, comments, indicators, and notification intervals. |
| `/api/rift-chart/*` | Internal aggregate champion/stat analytics | Strong inference | Public metadata has tiers, versions, tags, and old chart versions; Riot does not provide chart aggregates directly. |

## Unknowns

| Unknown | Why it remains unknown |
| --- | --- |
| Exact Riot application rate limits used by your.gg | Requires their backend/account context or successful API responses with headers. |
| Whether your.gg collects only searched players or also a larger seed corpus | Not visible from public passive evidence. |
| Exact scoring formula for OP/S/A/B/C/D/F | Client exposes thresholds from numeric score to label, but not the backend score formula. |
| Whether augment stats come from official Match-V5 fields, timeline events, local client integration, or a hybrid | Public endpoint is empty and no raw match payloads were fetched. |
| Paid PlayReport model inputs and prompts | Behind account/subscription flows and not accessed. |
