# Social Moderation Platform — Rules Engine & Recommendations

## Overview 🧭

This repository centers on the Drools rules engine that powers recommendations and user moderation. The frontend and Spring Boot app are present but secondary; the core logic lives in the rules and facts.

Key modules:
- `kjar-example/facts` — fact model (POJOs) used by rules.
- `kjar-example/drools-spring-kjar` — Drools rules (.drl) packaged as a KJAR.
- `kjar-example/drools-spring-app` — minimal Spring Boot integration that loads and executes the KJAR.
- `frontend/` — Angular UI (optional for understanding the engine).

Rules location:
- `kjar-example/drools-spring-kjar/src/main/resources/sbnz/integracija/*.drl`
  - `classify-item-rules.drl`
  - `feed-friends-rules.drl`
  - `feed-recommend-rules.drl`
  - `user-moderation-rules.drl`

## Facts (data model) 📚

Available compiled facts under `demo.facts` include:
- Core feed/recommendation facts: `RecommendedFeedRequest`, `FriendIds`, `CandidatePost`, `PostFact`, `PopularHashtag`, `PopularPost`, `UserFeedContext`, `UserAuthoredCount`, `SimilarUser`, `PostLikers`, `UserLikedPosts`.
- Moderation facts: `moderation.UserInfo`, `moderation.ReportEvent`, `moderation.BlockEvent`, `moderation.ModerationFlag`.
- Utility/domain: `BlockedIds`, and `Item` with `Item.Category` for a simple classification example.

Note: Source code resides in `kjar-example/facts` (package `demo.facts`); compiled classes are visible under `kjar-example/facts/target/classes/demo/facts`.

## Recommendation engine 🎯

The recommendation logic is implemented in `feed-recommend-rules.drl` and uses agenda groups and a scoring approach over `CandidatePost` facts.

Inputs (insert as facts):
- `RecommendedFeedRequest` (userId)
- `FriendIds` and `UserAuthoredCount` (to route the strategy)
- Candidate universe: multiple `CandidatePost(post)` facts
- Context: `UserFeedContext` (liked/authored hashtags), `PopularPost`, `PopularHashtag`, `SimilarUser`, `PostLikers`, `UserLikedPosts`

Globals:
- `NOW: LocalDateTime` — time anchor for recency rules
- `recommendFeedOut: List` — output collector for scored candidates

Strategy router (agenda-group: `feed-recommend-router`):
- Insert `UseBase` when the user has friends or authored posts; otherwise insert `UseNew`.

Scoring (agenda-group: `feed-recommend-score`):
- Base rules (`UseBase`):
  - Recent posts (<24h)
  - Posts containing liked or authored hashtags
  - Popular posts and popular hashtags
  - Boost when a hashtag is both popular and user-liked
- New-user rules (`UseNew`):
  - Posts liked by similar users (similarity ≥ 0.5)
  - Posts with high overlap of likers with the user’s previously liked posts (≥ 0.7)
- Guard: remove candidates authored by the user or their friends (recommendation list aims beyond friend/self content).

Output (agenda-group: `feed-recommend-output`):
- Any `CandidatePost` with `score > 0` is appended to the `recommendFeedOut` global list.

## Friends feed rules 👥

Implemented in `feed-friends-rules.drl` (agenda-group `feed-friends-select`).
- Selects posts authored by friends in the last 24h.
- Excludes authors present in `BlockedIds`.
- Uses `friendsFeedOut: List` global to collect `PostFact` results.

## User moderation rules 🛡️

Implemented in `user-moderation-rules.drl` (agenda-group `user-moderation`). Uses event processing with sliding time windows:
- >5 `ReportEvent`s in 24h → suspend posting for 24h.
- >8 reports in 48h → suspend posting for 48h.
- >4 `BlockEvent`s in 24h → suspend posting for 24h.
- ≥2 blocks in 48h AND ≥4 reports in 24h → suspend login for 48h.
- >3 blocks in 6h → suspend posting for 12h (rapid escalation).
- >12 reports in 7d → suspend login for 72h (chronic behavior).

Each rule emits a `ModerationFlag` into the `moderationFlags: List` global, including the user id, reason, scope (`POSTING` or `LOGIN`), and an expiry timestamp.

## End-to-end flow 🔄

1. Create a `KieSession` from the KJAR (classpath or Maven GAV).
2. Set required globals (`NOW`, output lists like `recommendFeedOut`, `friendsFeedOut`, `moderationFlags`).
3. Insert facts: requests, context, candidates, and any relevant events.
4. Focus agenda groups as needed (e.g., validate → router → score → output).
5. Fire rules and read outputs from the global lists.

## Build the KJAR ⚒️

```powershell
cd kjar-example\facts
mvn clean install

cd ..\drools-spring-kjar
mvn clean package
```

Artifacts are produced under `kjar-example/drools-spring-kjar/target`.

## Tuning knobs ⚙️

- Thresholds: similarity (≥0.5), likers overlap (≥0.7), recency window (24h)
- Popularity signals: presence of `PopularPost` and `PopularHashtag`
- Scoring: each matched condition adds `+1` via `CandidatePost.addScore(...)`
- Agenda control: use agenda-groups to orchestrate validation → routing → scoring → output
- Event windows: moderation uses `over window:time(...)` for 6h, 24h, 48h, 7d

## Testing 🧪

- Run all rule tests and builds:

```powershell
cd kjar-example
mvn clean package
```

- Surefire reports are generated under each module’s `target/surefire-reports`.

## Minimal notes on app/UI 📉

- Spring Boot (`drools-spring-app`) provides a thin runtime for the KJAR.
- The Angular app is optional for understanding rules; it can call backend `/api` routes via a proxy.
