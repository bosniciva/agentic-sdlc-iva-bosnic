# GAMEPLAY-STORY-001: Move PacMan through the maze

**Priority**: Core (Pareto 20%)

**Architecture References**:
- `01-introduction-and-goals.md` §"Key Requirements" items 1, 2
- `05-building-block-view.md` §"Component View - Game Client"; §"Building Block Responsibilities" — Game Engine, Maze, Input Handler, Renderer
- `06-runtime-view.md` §"Play a game and submit the score"
- `08-cross-cutting-concepts.md` §"Input and Score Validation"; §"Logging"
- `09-architecture-decisions.md` ADR-002 "Run the game loop client-side"
- `10-quality-requirements.md` Q1
- `11-risks-and-technical-debts.md` R6

---

## Open Questions

**OQ-001** — Sign-in and game-state loss on redirect

`06-runtime-view.md` §"Log in with Google Sign-In" states that the API Client "redirects the player to Google Sign-In using OAuth2/OIDC." `09-architecture-decisions.md` ADR-002 §"Consequences" states that "game state is lost on reload." A full-page redirect navigates away from the SPA, which would lose any in-progress game. It is not clear from the architecture whether the sign-in flow uses a full-page redirect (losing game state), a pop-up window, or a silent token refresh.

This question does not affect GAMEPLAY-STORY-001 (movement does not involve sign-in), but it must be resolved before LEADERBOARD-STORY-001 is written. Sign-in scenarios in this bundle are written neutrally where they appear.

**Decision needed**: Does the API Client use a full-page redirect, a pop-up, or another mechanism for Google Sign-In?

---

## Original Story

AS A player
I WANT to move PacMan through the maze using the keyboard, with the Maze enforcing walls and the Renderer drawing each frame on the Canvas
SO THAT I can navigate the game and play without any network round trips during gameplay

### SCENARIO 1: PacMan moves in a valid direction

**Scenario ID**: GAMEPLAY-STORY-001-S1

**GIVEN**
* The Game Client SPA is loaded in a desktop browser
* A game is in progress and PacMan is on a corridor tile with an open tile to the right

**WHEN**
* The player presses the ArrowRight key and the Game Engine processes the next frame

**THEN**
* PacMan's tile position advances exactly one tile to the right
* The Renderer draws the updated position on the HTML5 Canvas
* 0 network requests are made during this frame

### SCENARIO 2: PacMan is blocked by a wall

**Scenario ID**: GAMEPLAY-STORY-001-S2

**GIVEN**
* A game is in progress and PacMan is on a corridor tile adjacent to a wall tile in the right direction

**WHEN**
* The player presses the ArrowRight key and the Game Engine processes the next frame

**THEN**
* PacMan's tile position is unchanged
* The Renderer draws the same position on the Canvas
* The game loop continues without error

### SCENARIO 3: An unsupported key is ignored

**Scenario ID**: GAMEPLAY-STORY-001-S3

**GIVEN**
* A game is in progress and PacMan's last movement intent was "right"

**WHEN**
* The player presses a key that the Input Handler does not map to a movement intent (for example the "z" key)

**THEN**
* PacMan's movement intent remains "right"
* PacMan's position is unchanged from what the current intent would produce
* No error is thrown and the game loop continues

### SCENARIO 4: Frame time stays within the 16.7 ms budget (Q1 / R6)

**Scenario ID**: GAMEPLAY-STORY-001-S4

**GIVEN**
* A game is running in a desktop browser on the course workstation
* PacMan has been moving continuously for at least 60 consecutive frames

**WHEN**
* The Renderer completes each of those frames

**THEN**
* Each frame completes in ≤ 16.7 ms
* The sustained frame rate is 60 FPS
* 0 network requests are made during those 60 frames

---

## INFRA Sub-Stories

### GAMEPLAY-INFRA-001.1: Build and serve the Game Client as the `web` Docker Compose service

**Architecture References**:
- `07-deployment-view.md` §"Local Development"; §"CI Environment"; §"Production Environment"; §"Container Mapping" (`web` row)
- `09-architecture-decisions.md` ADR-001 §"Packaging and CI/CD"; ADR-005 "Deploy production as a single Docker VM"
- `02-constraints.md` §"Technical Constraints" — Docker, GitHub Actions, Node 20, React 18

AS A developer
I WANT the Game Client SPA built by a Node 20 Dockerfile and served by the `web` Compose service (Caddy)
SO THAT the game is reachable locally with one command and the same image is deployable in production without further changes

#### SCENARIO 1: `web` service starts and serves the SPA locally (Q4)

**Scenario ID**: GAMEPLAY-INFRA-001.1-S1

**GIVEN**
* Docker and Docker Compose are installed on a clean machine
* The repository is checked out

**WHEN**
* The developer runs `docker compose up`

**THEN**
* The `web` service reaches healthy status within 60 seconds
* A GET request to `http://localhost` returns HTTP 200 with a response body that contains the React SPA HTML
* No database server is installed or started on the host machine

#### SCENARIO 2: GitHub Actions builds the SPA and the `web` image

**Scenario ID**: GAMEPLAY-INFRA-001.1-S2

**GIVEN**
* A push or pull request triggers the GitHub Actions pipeline

**WHEN**
* The Node 20 build step runs

**THEN**
* The step exits with code 0 and the built static assets are present
* A failing build step exits with a non-zero code and no image is published

#### SCENARIO 3: Caddy forwards API calls and serves the SPA in production

**Scenario ID**: GAMEPLAY-INFRA-001.1-S3

**GIVEN**
* The `web` service is running on a Linux VM with a registered domain name and TLS configured

**WHEN**
* A browser requests the SPA root URL over HTTPS

**THEN**
* Caddy terminates TLS and returns HTTP 200 with the SPA HTML
* A request to `/api/*` is forwarded to the `api` service and not served as a static file

---

### GAMEPLAY-INFRA-001.2: Configure Vitest for game-logic tests in CI without a browser or canvas

**Architecture References**:
- `07-deployment-view.md` §"CI Environment"
- `08-cross-cutting-concepts.md` §"Testing Approach"
- `09-architecture-decisions.md` ADR-001 §"Testing" row; ADR-002 §"Consequences" ("Game logic must be written in TypeScript and kept separate from rendering")
- `10-quality-requirements.md` Q2
- `02-constraints.md` §"Technical Constraints" — Vitest, Node 20

AS A developer
I WANT Vitest configured to run Game Engine, Maze, Input Handler, and Renderer unit tests on Node 20 in CI with no browser, canvas, or API process
SO THAT every pull request verifies game-logic correctness before an image is published, as required by Q2

#### SCENARIO 1: Vitest runs game-logic tests with no browser or API in CI (Q2)

**Scenario ID**: GAMEPLAY-INFRA-001.2-S1

**GIVEN**
* The GitHub Actions pipeline is running
* No browser and no Game API process have been started

**WHEN**
* The Vitest step executes

**THEN**
* All game-logic tests (Game Engine, Maze, Input Handler) run and the step exits with code 0
* 0 browser processes are started
* 0 network calls are made during the test run

#### SCENARIO 2: A failing game-logic test blocks the pipeline

**Scenario ID**: GAMEPLAY-INFRA-001.2-S2

**GIVEN**
* At least one Vitest test for Game Engine, Maze, or Input Handler is failing

**WHEN**
* The Vitest step executes in the pipeline

**THEN**
* The step exits with a non-zero code
* No Docker image is published

---

INFRA Events: N/A — the Gameplay context has no message queues or event topics; all state is local to the browser session (`05-building-block-view.md` §"Bounded Contexts").

INFRA Monitoring: The Game Client writes only errors to the browser console during play and makes no server-side log entries during the game loop (`08-cross-cutting-concepts.md` §"Logging"). No server-side monitoring story applies to the Gameplay context.

---

## BE Sub-Stories

BE: N/A — the Gameplay context runs entirely in the Game Client (browser). There is no server-side component in the Gameplay bounded context (`05-building-block-view.md` §"Bounded Contexts"; `09-architecture-decisions.md` ADR-002).

---

## FE Sub-Stories

### GAMEPLAY-FE-001.1: Implement the Game Engine — frame loop coordinating Maze, Input Handler, and Renderer

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Game Engine, Maze, Input Handler, Renderer
- `06-runtime-view.md` §"Play a game and submit the score"
- `08-cross-cutting-concepts.md` §"Input and Score Validation"
- `09-architecture-decisions.md` ADR-002
- `10-quality-requirements.md` Q1, Q2
- `11-risks-and-technical-debts.md` R6

AS A developer
I WANT the Game Engine to run a frame loop that reads the Input Handler, advances game state through the Maze, and calls the Renderer each frame, entirely within the browser
SO THAT PacMan can be moved without any network calls, satisfying requirement 2 and Q1

#### SCENARIO 1: Game Engine processes one frame — valid move

**Scenario ID**: GAMEPLAY-FE-001.1-S1

**GIVEN**
* A Game Engine is initialised with a Maze layout and PacMan at a starting corridor tile (Vitest environment, no browser or canvas)
* The Input Handler reports intent "right"
* The tile to the right of PacMan is a walkable corridor

**WHEN**
* `GameEngine.tick()` is called once

**THEN**
* PacMan's tile position is the corridor tile one step to the right of the starting tile
* `Renderer.drawFrame()` is called exactly once with the updated game state
* No network call is made

#### SCENARIO 2: Game Engine processes one frame — wall blocks move

**Scenario ID**: GAMEPLAY-FE-001.1-S2

**GIVEN**
* PacMan is on a corridor tile adjacent to a wall tile to the right
* The Input Handler reports intent "right"

**WHEN**
* `GameEngine.tick()` is called once

**THEN**
* PacMan's tile position is unchanged
* `Renderer.drawFrame()` is called once with the unchanged position

---

### GAMEPLAY-FE-001.2: Implement the Maze component — fixed layout and walkability rules

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Maze
- `08-cross-cutting-concepts.md` §"Input and Score Validation"
- `09-architecture-decisions.md` ADR-002
- `10-quality-requirements.md` Q2

AS A developer
I WANT the Maze to hold the single fixed layout and expose walkability checks to the Game Engine
SO THAT PacMan can only move through valid corridors, as required by requirement 1

#### SCENARIO 1: Maze reports a corridor tile as walkable

**Scenario ID**: GAMEPLAY-FE-001.2-S1

**GIVEN**
* The Maze is initialised from the fixed layout (Vitest environment)
* A known corridor tile coordinate is used

**WHEN**
* `Maze.isWalkable(corridorTile)` is called

**THEN**
* The return value is `true`

#### SCENARIO 2: Maze reports a wall tile as not walkable

**Scenario ID**: GAMEPLAY-FE-001.2-S2

**GIVEN**
* The Maze is initialised from the fixed layout
* A known wall tile coordinate is used

**WHEN**
* `Maze.isWalkable(wallTile)` is called

**THEN**
* The return value is `false`

---

### GAMEPLAY-FE-001.3: Implement the Input Handler — keyboard events to movement intents

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Input Handler
- `08-cross-cutting-concepts.md` §"Input and Score Validation"
- `09-architecture-decisions.md` ADR-002
- `10-quality-requirements.md` Q2

AS A developer
I WANT the Input Handler to map ArrowUp, ArrowDown, ArrowLeft, and ArrowRight to movement intents and silently ignore all other keys
SO THAT the player can control PacMan and unsupported keys have no effect, as required by requirement 1 and Chapter 8 §"Input and Score Validation"

#### SCENARIO 1: Arrow key maps to a movement intent

**Scenario ID**: GAMEPLAY-FE-001.3-S1

**GIVEN**
* The Input Handler is initialised (Vitest environment)

**WHEN**
* A `keydown` event for `ArrowRight` is dispatched

**THEN**
* `InputHandler.currentIntent()` returns `"right"`

#### SCENARIO 2: Unsupported key leaves the intent unchanged

**Scenario ID**: GAMEPLAY-FE-001.3-S2

**GIVEN**
* `InputHandler.currentIntent()` returns `"right"`

**WHEN**
* A `keydown` event for `"z"` is dispatched

**THEN**
* `InputHandler.currentIntent()` still returns `"right"`

---

### GAMEPLAY-FE-001.4: Implement the Renderer — draw game state to the HTML5 Canvas

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Renderer
- `08-cross-cutting-concepts.md` §"Logging" (errors only to console during play)
- `09-architecture-decisions.md` ADR-001 §"Game loop and rendering"; ADR-002
- `10-quality-requirements.md` Q1
- `11-risks-and-technical-debts.md` R6

AS A developer
I WANT the Renderer to draw the current game state to the HTML5 Canvas each frame without making network calls
SO THAT the player sees the maze and PacMan position and the 60 FPS / 16.7 ms frame budget is not broken by I/O, as required by Q1 and R6

#### SCENARIO 1: Renderer draws a frame without throwing errors

**Scenario ID**: GAMEPLAY-FE-001.4-S1

**GIVEN**
* A Canvas 2D context mock is provided (Vitest environment)
* A game state with a valid Maze layout and PacMan at a corridor tile is available

**WHEN**
* `Renderer.drawFrame(gameState, context)` is called once

**THEN**
* No exception is thrown
* At least one drawing call (`fillRect` or equivalent) is made on the context mock

#### SCENARIO 2: Renderer makes no network calls across 60 frames

**Scenario ID**: GAMEPLAY-FE-001.4-S2

**GIVEN**
* A Canvas 2D context mock and a network-call recorder are provided

**WHEN**
* `Renderer.drawFrame(gameState, context)` is called 60 times

**THEN**
* The network-call recorder reports 0 calls

---

## Implementation Order

```
GAMEPLAY-INFRA-001.1   (web Compose service, Node 20 build, Caddy)
GAMEPLAY-INFRA-001.2   (Vitest in CI, no browser/canvas)
       ↓
GAMEPLAY-FE-001.2      (Maze — layout and walkability)
GAMEPLAY-FE-001.3      (Input Handler — keyboard to intents)
GAMEPLAY-FE-001.4      (Renderer — Canvas draw, no network calls)
       ↓
GAMEPLAY-FE-001.1      (Game Engine — integrates Maze, Input Handler, Renderer)
       ↓
GAMEPLAY-STORY-001     (end-to-end: player moves PacMan in a browser,
                        frame rate ≤ 16.7 ms, 0 network calls during play)
```

---

# GAMEPLAY-STORY-002: Eat pellets and earn points

**Priority**: Core (Pareto 20%)

**Architecture References**:
- `01-introduction-and-goals.md` §"Key Requirements" items 1, 2
- `05-building-block-view.md` §"Building Block Responsibilities" — Maze ("Holds the layout, pellets, and walkability rules"), Scoring ("Tracks pickups and collisions and computes the final score"), Renderer ("Draws the current game state to the HTML5 Canvas")
- `05-building-block-view.md` §"Bounded Contexts" — Gameplay model includes "pellet, game score"
- `06-runtime-view.md` §"Play a game and submit the score"
- `08-cross-cutting-concepts.md` §"Input and Score Validation"
- `09-architecture-decisions.md` ADR-002 "Run the game loop client-side"
- `10-quality-requirements.md` Q1, Q2
- `11-risks-and-technical-debts.md` R6
- `12-glossary.md` §"Domain Terms" — Pellet ("A small item placed throughout the maze that PacMan eats to gain points"), Power pellet ("A larger pellet that temporarily allows PacMan to eat ghosts")

---

## Architecture Gap

**GAP-001 — Point values are not defined in the architecture.**

The architecture confirms that eating pellets gains points (Ch12 §"Domain Terms", Pellet definition) and that Scoring "computes the final score" (Ch5 §"Building Block Responsibilities"), but no numeric point values are specified anywhere in the architecture files. The scenarios below use the named constants `PELLET_POINTS` and `POWER_PELLET_POINTS` in place of concrete numbers.

**Decision needed**: What is the point value for a standard pellet, and what is the point value for a power pellet? These values must be defined before the Scoring component can be implemented and before the Score Validation plausibility check (Q6, LEADERBOARD-STORY-004) can set a maximum score bound.

---

## Original Story

AS A player
I WANT PacMan to collect pellets as I move through the maze, with each pellet removed from the maze and its points added to my score
SO THAT I accumulate a score during play that reflects how many pellets I collected

### SCENARIO 1: PacMan eats a pellet — score increases and pellet is removed

**Scenario ID**: GAMEPLAY-STORY-002-S1

**GIVEN**
* A game is in progress
* PacMan is on a corridor tile adjacent to a tile containing a standard pellet

**WHEN**
* The player moves PacMan onto the pellet tile and the Game Engine processes that frame

**THEN**
* The pellet is removed from the Maze (the tile no longer contains a pellet)
* Scoring adds `PELLET_POINTS` to the current score
* The Renderer draws the updated score and the now-empty tile in the same frame
* 0 network requests are made

### SCENARIO 2: PacMan walks over an empty tile — score is unchanged

**Scenario ID**: GAMEPLAY-STORY-002-S2

**GIVEN**
* A game is in progress
* PacMan is adjacent to a corridor tile that contains no pellet (already collected or never had one)
* The current score is S

**WHEN**
* The player moves PacMan onto that empty tile and the Game Engine processes the frame

**THEN**
* The score remains S (unchanged)
* The Maze tile state is unchanged

---

## INFRA Sub-Stories

INFRA Deployment: covered by GAMEPLAY-INFRA-001.1 — the `web` Compose service and Node 20 build that serves the SPA apply unchanged to this story.

INFRA Data store: N/A — pellet state and score are in-memory game state inside the Game Client; they are not persisted (`05-building-block-view.md` §"Bounded Contexts": "Lives only in the browser during a game; not persisted").

INFRA Events: N/A — the Gameplay context has no message queues or event topics (`05-building-block-view.md` §"Bounded Contexts").

INFRA Monitoring: covered by GAMEPLAY-INFRA-001.2 — Vitest is already configured in CI. Scoring is explicitly listed in Q2 ("Vitest tests for Game Engine, Maze, Ghost AI, **and Scoring** execute and pass"), so the Scoring tests run under the existing Vitest setup. No new INFRA sub-story is needed.

---

## BE Sub-Stories

BE: N/A — Scoring and Maze pellet tracking run entirely in the Game Client browser. There is no server-side component in the Gameplay bounded context (`05-building-block-view.md` §"Bounded Contexts"; `09-architecture-decisions.md` ADR-002).

---

## FE Sub-Stories

### GAMEPLAY-FE-002.1: Implement Scoring — pellet point accumulation and current score

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Scoring ("Tracks pickups and collisions and computes the final score")
- `06-runtime-view.md` §"Play a game and submit the score"
- `08-cross-cutting-concepts.md` §"Input and Score Validation"
- `09-architecture-decisions.md` ADR-002
- `10-quality-requirements.md` Q1, Q2
- `11-risks-and-technical-debts.md` R6

AS A developer
I WANT Scoring to add `PELLET_POINTS` to the current score each time PacMan collects a pellet, and to expose `currentScore()` for the Renderer and `finalScore()` for the API Client
SO THAT the score accurately reflects the player's pickups and can be displayed and eventually submitted

#### SCENARIO 1: Collecting a pellet adds PELLET_POINTS (Q2)

**Scenario ID**: GAMEPLAY-FE-002.1-S1

**GIVEN**
* A `Scoring` instance is initialised with score 0 (Vitest environment, no browser or canvas)

**WHEN**
* `Scoring.onPelletCollected()` is called once

**THEN**
* `Scoring.currentScore()` returns exactly `PELLET_POINTS`

#### SCENARIO 2: Collecting N pellets accumulates N × PELLET_POINTS (Q2)

**Scenario ID**: GAMEPLAY-FE-002.1-S2

**GIVEN**
* A `Scoring` instance is initialised with score 0

**WHEN**
* `Scoring.onPelletCollected()` is called N times (N > 1)

**THEN**
* `Scoring.currentScore()` returns exactly `N × PELLET_POINTS`

#### SCENARIO 3: Collecting a power pellet adds POWER_PELLET_POINTS (Q2)

**Scenario ID**: GAMEPLAY-FE-002.1-S3

**GIVEN**
* A `Scoring` instance is initialised with score 0

**WHEN**
* `Scoring.onPowerPelletCollected()` is called once

**THEN**
* `Scoring.currentScore()` returns exactly `POWER_PELLET_POINTS`

#### SCENARIO 4: Score computation runs without canvas or browser (Q2)

**Scenario ID**: GAMEPLAY-FE-002.1-S4

**GIVEN**
* The Vitest suite runs on Node 20 with no browser, canvas, or API process started

**WHEN**
* All `Scoring` unit tests execute

**THEN**
* All tests pass and exit with code 0
* 0 browser processes are started
* 0 network calls are made

---

### GAMEPLAY-FE-002.2: Extend Maze pellet tracking — collection and duplicate-collection guard

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Maze ("Holds the layout, pellets, and walkability rules")
- `08-cross-cutting-concepts.md` §"Input and Score Validation"
- `09-architecture-decisions.md` ADR-002
- `10-quality-requirements.md` Q2

AS A developer
I WANT the Maze to remove a pellet when PacMan collects it and to prevent the same pellet being collected twice
SO THAT Scoring receives exactly one `onPelletCollected()` call per pellet and the displayed maze reflects the correct remaining pellets

#### SCENARIO 1: Collecting a pellet removes it from the Maze (Q2)

**Scenario ID**: GAMEPLAY-FE-002.2-S1

**GIVEN**
* A `Maze` instance is initialised from the fixed layout (Vitest environment)
* Tile T is a standard pellet tile: `Maze.hasPellet(T)` returns `true`

**WHEN**
* `Maze.collectPellet(T)` is called

**THEN**
* `Maze.hasPellet(T)` returns `false`
* `Maze.pelletsRemaining()` decreases by exactly 1

#### SCENARIO 2: The same pellet cannot be collected twice (Q2)

**Scenario ID**: GAMEPLAY-FE-002.2-S2

**GIVEN**
* `Maze.collectPellet(T)` has already been called for tile T

**WHEN**
* `Maze.collectPellet(T)` is called a second time

**THEN**
* `Maze.hasPellet(T)` remains `false`
* `Maze.pelletsRemaining()` does not decrease further
* No exception is thrown

#### SCENARIO 3: Walking over an empty tile does not trigger a pellet event (Q2)

**Scenario ID**: GAMEPLAY-FE-002.2-S3

**GIVEN**
* Tile T is an empty corridor tile: `Maze.hasPellet(T)` returns `false`

**WHEN**
* The Game Engine moves PacMan onto tile T and checks `Maze.hasPellet(T)`

**THEN**
* `Maze.hasPellet(T)` returns `false`
* `Scoring.onPelletCollected()` is not called

---

### GAMEPLAY-FE-002.3: Extend the Renderer to display the current score

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Renderer ("Draws the current game state to the HTML5 Canvas")
- `09-architecture-decisions.md` ADR-001 §"Game loop and rendering"; ADR-002
- `10-quality-requirements.md` Q1
- `11-risks-and-technical-debts.md` R6

AS A player
I WANT the Renderer to display the current score on the Canvas each frame
SO THAT I can see my score change as I collect pellets, without any network calls

#### SCENARIO 1: Renderer includes the current score in the frame (Q1)

**Scenario ID**: GAMEPLAY-FE-002.3-S1

**GIVEN**
* A Canvas 2D context mock is provided (Vitest environment)
* A game state with `currentScore` set to a value V > 0 is available

**WHEN**
* `Renderer.drawFrame(gameState, context)` is called

**THEN**
* At least one drawing call on the context mock includes or references the value V (for example via `fillText`)
* No exception is thrown
* 0 network calls are made

---

## Implementation Order

```
GAMEPLAY-INFRA-001.1 (reference — web service already set up)
GAMEPLAY-INFRA-001.2 (reference — Vitest already configured; Scoring tests
                      will run under the existing setup per Q2)
       ↓
GAMEPLAY-FE-002.2    (Maze pellet tracking — extend from GAMEPLAY-FE-001.2)
GAMEPLAY-FE-002.1    (Scoring — pellet accumulation and currentScore/finalScore)
GAMEPLAY-FE-002.3    (Renderer score display — extend from GAMEPLAY-FE-001.4)
       ↓
GAMEPLAY-STORY-002   (end-to-end: player collects pellets in a browser,
                      score increments by PELLET_POINTS per pellet,
                      pellet removed from maze, 0 network calls)
```
