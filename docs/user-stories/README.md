# User Story Inventory

## Bounded Contexts

| Context | Domain ID | Owner |
|---------|-----------|-------|
| Gameplay | GAMEPLAY | Game Client (React 18 / TypeScript) |
| Leaderboard | LEADERBOARD | Game API + Game Database (Python / FastAPI / SQLite) |
| Identity | — | External (Google Sign-In) — no stories |

---

## Story Inventory

| Story ID | Title | Domain | Priority | Status | Bundle |
|----------|-------|--------|----------|--------|--------|
| GAMEPLAY-STORY-001 | Move PacMan through the maze | GAMEPLAY | Core | Approved | [gameplay.md](./gameplay.md) |
| GAMEPLAY-STORY-002 | Eat pellets and earn points | GAMEPLAY | Core | Approved | [gameplay.md](./gameplay.md) |
| LEADERBOARD-STORY-001 | Submit a score | LEADERBOARD | Core | Approved | [leaderboard.md](./leaderboard.md) |
| GAMEPLAY-STORY-003 | Ghosts chase PacMan and a collision costs a life | GAMEPLAY | Supporting | Not started | [gameplay.md](./gameplay.md) |
| GAMEPLAY-STORY-004 | Power pellet makes ghosts vulnerable | GAMEPLAY | Supporting | Not started | [gameplay.md](./gameplay.md) |
| GAMEPLAY-STORY-005 | Game over with final score | GAMEPLAY | Supporting | Not started | [gameplay.md](./gameplay.md) |
| LEADERBOARD-STORY-002 | View the leaderboard | LEADERBOARD | Supporting | Not started | [leaderboard.md](./leaderboard.md) |
| LEADERBOARD-STORY-003 | Reject an invalid or missing ID token | LEADERBOARD | Supporting | Not started | [leaderboard.md](./leaderboard.md) |
| LEADERBOARD-STORY-004 | Reject a plausibility-failing score | LEADERBOARD | Supporting | Not started | [leaderboard.md](./leaderboard.md) |
| GAMEPLAY-STORY-006 | Display submission result to the player | GAMEPLAY | Supporting | Not started | [gameplay.md](./gameplay.md) |

---

## Pareto Analysis

### Why these three stories are Core

**GAMEPLAY-STORY-001**, **GAMEPLAY-STORY-002**, and **LEADERBOARD-STORY-001** are the Core stories (3 of 10 = 30%, rounded up from 20% minimum of 2; justified below).

**GAMEPLAY-STORY-001** — Move PacMan through the maze: required for MVP (Maze, Game Engine, Input Handler, and Renderer must exist before any other Gameplay story); all five remaining Gameplay stories depend on it; exercises four Gameplay components; primary player value.

**GAMEPLAY-STORY-002** — Eat pellets and earn points: movement alone produces no score, so LEADERBOARD-STORY-001 (Core) cannot be completed without pellet-scoring existing. GAMEPLAY-STORY-001 without pellets is not a playable game. Scores the most points that reach the leaderboard; depends only on GAMEPLAY-STORY-001 (also Core), satisfying the rule that a core story must not depend on a supporting story.

**LEADERBOARD-STORY-001** — Submit a score: MVP blocker for all Leaderboard functionality; absorbs the Google Sign-In flow (OIDC/API Client) so it has no dependency on a supporting story; exercises Score Endpoint, Auth Verification, Score Validation, Repository, Identity Provider, and API Client. Its INFRA/BE/FE sub-stories can be built and tested from a submitted score payload independently of the Gameplay stories; only the end-to-end scenario that starts from a real played game depends on GAMEPLAY-STORY-005.

Going to 3 Core (30%) is justified because GAMEPLAY-STORY-002 is a direct structural prerequisite of LEADERBOARD-STORY-001: without a Scoring component, no final score exists to submit.

### Architecture references per story

| Story ID | `architecture/` file + exact heading or ID |
|----------|---------------------------------------------|
| GAMEPLAY-STORY-001 | `01` §"Key Requirements"; `05` §"Component View - Game Client", §"Building Block Responsibilities"; `06` §"Play a game and submit the score"; `08` §"Input and Score Validation"; ADR-002; Q1 |
| GAMEPLAY-STORY-002 | `01` §"Key Requirements"; `05` §"Building Block Responsibilities"; `06` §"Play a game and submit the score"; `08` §"Input and Score Validation"; ADR-002; Q1, Q2 |
| GAMEPLAY-STORY-003 | `01` §"Key Requirements"; `05` §"Building Block Responsibilities"; `06` §"Play a game and submit the score"; ADR-002; Q1, Q2 |
| GAMEPLAY-STORY-004 | `01` §"Key Requirements"; `05` §"Building Block Responsibilities"; `06` §"Play a game and submit the score"; ADR-002; Q1 |
| GAMEPLAY-STORY-005 | `01` §"Key Requirements"; `05` §"Building Block Responsibilities"; `06` §"Play a game and submit the score"; ADR-002; Q1, Q2 |
| LEADERBOARD-STORY-001 | `01` §"Key Requirements"; `03` §"Business Context", §"Technical Context"; `05` §"Building Block Responsibilities"; `06` §"Log in with Google Sign-In", §"Play a game and submit the score"; `08` §"Authentication and Authorization", §"Error Handling", §"Logging", §"Input and Score Validation", §"API Contract and Versioning", §"Persistence Access", §"Configuration and Secrets"; ADR-001, ADR-003, ADR-004; Q3, Q5, Q6, Q7; R1, R3 |
| LEADERBOARD-STORY-002 | `01` §"Key Requirements"; `03` §"Business Context", §"Technical Context"; `05` §"Building Block Responsibilities"; `06` §"View the leaderboard without login"; `08` §"Persistence Access", §"API Contract and Versioning"; ADR-004; Q7 |
| LEADERBOARD-STORY-003 | `01` §"Key Requirements"; `05` §"Building Block Responsibilities"; `06` §"Play a game and submit the score"; `08` §"Authentication and Authorization", §"Error Handling"; ADR-003; Q5; R3 |
| LEADERBOARD-STORY-004 | `01` §"Key Requirements"; `05` §"Building Block Responsibilities"; `06` §"Play a game and submit the score"; `08` §"Input and Score Validation", §"Error Handling"; ADR-001 §"Consequences"; Q6; R1 |
| GAMEPLAY-STORY-006 | `03` §"Business Context"; `05` §"Building Block Responsibilities"; `06` §"Play a game and submit the score"; `08` §"Error Handling"; Q5, Q6 |

### Dependency chart

```
GAMEPLAY-STORY-001  [Core]
  ├─► GAMEPLAY-STORY-002  [Core]
  │     └─► GAMEPLAY-STORY-005  (game over + final score;
  │           also needs GAMEPLAY-STORY-003)
  └─► GAMEPLAY-STORY-003  [Supporting]
        └─► GAMEPLAY-STORY-005  [Supporting]
              └─► LEADERBOARD-STORY-001  [Core — end-to-end scenario only;
                    │                     INFRA/BE/FE sub-stories are independent]
                    ├─► LEADERBOARD-STORY-002  [Supporting]
                    ├─► LEADERBOARD-STORY-003  [Supporting]
                    ├─► LEADERBOARD-STORY-004  [Supporting]
                    └─► GAMEPLAY-STORY-006     [Supporting]

GAMEPLAY-STORY-004  [Supporting]
  depends on: GAMEPLAY-STORY-002 (Scoring must exist),
              GAMEPLAY-STORY-003 (Ghost AI must exist)
```

Notes:
- GAMEPLAY-STORY-005 depends on GAMEPLAY-STORY-002 and GAMEPLAY-STORY-003 only, not on GAMEPLAY-STORY-004.
- LEADERBOARD-STORY-001 INFRA/BE/FE sub-stories start from a score payload and can be built independently. Only its end-to-end scenario requires a real played game (GAMEPLAY-STORY-005 done).
- Q2 (game-logic testability without canvas/browser), Q3 (backend testability without browser), and Q4 (one-command startup) are covered as INFRA sub-stories inside the relevant bundles, not as standalone stories.

---

## Progress

📊 Pareto Progress: 3/3 core stories (100% of 20% core stories)
🎯 Core functionality coverage: ~80% of 80% target
