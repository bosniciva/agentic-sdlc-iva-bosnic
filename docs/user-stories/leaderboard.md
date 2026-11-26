# LEADERBOARD-STORY-001: Submit a score

**Priority**: Core (Pareto 20%)

**Architecture References**:
- `01-introduction-and-goals.md` §"Key Requirements" items 4, 5, 6, 7, 8
- `03-context-and-scope.md` §"Business Context" (row "Player – submitting a score"); §"Technical Context" (rows "Score submission", "Player sign-in", "ID token verification")
- `05-building-block-view.md` §"Component View - Game API"; §"Building Block Responsibilities" — Score Endpoint, Auth Verification, Score Validation, Repository, API Client; §"Player Data Stored"
- `06-runtime-view.md` §"Log in with Google Sign-In"; §"Play a game and submit the score"
- `07-deployment-view.md` §"Local Development"; §"CI Environment"; §"Production Environment"; §"Container Mapping"
- `08-cross-cutting-concepts.md` §"Authentication and Authorization"; §"Error Handling"; §"Logging"; §"Input and Score Validation"; §"API Contract and Versioning"; §"Persistence Access"; §"Configuration and Secrets"
- `09-architecture-decisions.md` ADR-001 §"Consequences"; ADR-003; ADR-004; ADR-005
- `10-quality-requirements.md` Q3, Q4, Q5, Q6, Q7
- `11-risks-and-technical-debts.md` R1, R3, R4, R8

---

## Open Questions

**OQ-001** — Sign-in mechanism and game-state loss on redirect (carried from GAMEPLAY-STORY-001)

`06-runtime-view.md` §"Log in with Google Sign-In" states that the API Client "redirects the player to Google Sign-In using OAuth2/OIDC." `09-architecture-decisions.md` ADR-002 §"Consequences" states "game state is lost on reload." A full-page redirect navigates away from the SPA, losing any in-progress game state. The sign-in scenarios in this bundle are written neutrally: "the player completes Google Sign-In and returns to the SPA with a valid ID token" — without specifying whether this uses a full-page redirect, a pop-up window, or a silent token refresh.

**Decision needed before LEADERBOARD-FE-001.1 is implemented**: Does the API Client use a full-page redirect, a pop-up, or another mechanism for the Google Sign-In flow?

---

## Independence Note

The INFRA, BE, and FE sub-stories in this bundle start from a submitted score payload (a valid or invalid ID token, a score value, and a play time). They do not require a played game to be built or running. Only the original story's end-to-end happy-path scenario (LEADERBOARD-STORY-001-S1) depends on GAMEPLAY-STORY-005 (Game over with final score) being complete.

---

## Architecture Gap Reference

**GAP-001** (raised in GAMEPLAY-STORY-002): No numeric point values are defined in the architecture. The "maximum possible score" referenced in Ch1 §"Key Requirements" item 6 and Q6 cannot therefore be given a concrete number in the Score Validation scenarios below. The scenarios use the named constant `MAX_SCORE` and `MIN_PLAY_TIME_SECONDS` in place of concrete bounds. These values must be decided before LEADERBOARD-BE-001.3 (Score Validation) can be implemented.

---

## Original Story

AS A player
I WANT to sign in with Google and submit my final score and play time to the API at the end of a game
SO THAT my score is recorded on the leaderboard and I can compete with other players

### SCENARIO 1: Player signs in and submits a valid score — score is stored (happy path, end-to-end)

**Scenario ID**: LEADERBOARD-STORY-001-S1

*Depends on GAMEPLAY-STORY-005 being complete (a real played game that produces a final score and play time).*

**GIVEN**
* The player has completed a game and GAMEPLAY-STORY-005 has produced a final score and play time
* The player completes Google Sign-In and returns to the SPA with a valid ID token (see OQ-001)
* The Game API is running with Google Sign-In configured

**WHEN**
* The API Client POSTs the final score, play time, and ID token to the Score Endpoint

**THEN**
* The Score Endpoint returns HTTP 201
* The Repository has written exactly one new row containing the player's Google subject ID, display name, score, and play time
* The player's email address is not stored
* The ID token is not stored
* The API log contains one line with method `POST`, the score submission path, status `201`, and a duration in ms — and does not contain the ID token or the request body

### SCENARIO 2: Player submits without signing in — submission is rejected

**Scenario ID**: LEADERBOARD-STORY-001-S2

**GIVEN**
* The Game API is running
* No ID token is present in the request

**WHEN**
* The API Client POSTs a score and play time to the Score Endpoint with no Authorization header

**THEN**
* The Score Endpoint returns HTTP 401 with a JSON body containing a `detail` field
* 0 rows are written to the database
* The API log records method `POST`, the score submission path, status `401`, and a duration — and does not contain any token value

---

## INFRA Sub-Stories

### LEADERBOARD-INFRA-001.1: Build and publish the `api` image in CI, blocked on failing tests

**Architecture References**:
- `07-deployment-view.md` §"CI Environment"; §"Container Mapping" (`api` row)
- `09-architecture-decisions.md` ADR-001 §"Packaging and CI/CD"; ADR-005
- `02-constraints.md` §"Technical Constraints" — Docker, GitHub Actions

AS A developer
I WANT the Game API Docker image built and published to GitHub Container Registry by GitHub Actions only after all pytest and pytest-bdd tests pass
SO THAT no broken image is ever published, satisfying requirement 7

#### SCENARIO 1: All tests pass — image is published

**Scenario ID**: LEADERBOARD-INFRA-001.1-S1

**GIVEN**
* A push or pull request triggers the GitHub Actions pipeline
* All pytest and pytest-bdd tests exit with code 0

**WHEN**
* The build and publish step runs

**THEN**
* The `api` Docker image is pushed to GitHub Container Registry
* The image tag matches the commit SHA or branch name

#### SCENARIO 2: A failing test blocks image publication

**Scenario ID**: LEADERBOARD-INFRA-001.1-S2

**GIVEN**
* At least one pytest or pytest-bdd test is failing

**WHEN**
* The pipeline runs

**THEN**
* The pipeline exits with a non-zero code before the publish step
* 0 images are pushed to GitHub Container Registry

---

### LEADERBOARD-INFRA-001.2: One-command local start of `api` and `web` services (Q4)

**Architecture References**:
- `07-deployment-view.md` §"Local Development"; §"Container Mapping"
- `09-architecture-decisions.md` ADR-004 ("database file lives on a Docker volume in local development"); ADR-005
- `10-quality-requirements.md` Q4

AS A developer
I WANT both the `api` and `web` Docker Compose services to start with one command on a clean machine with no database server installed
SO THAT any developer can run the full system locally immediately after checkout, satisfying Q4

#### SCENARIO 1: Both services reach healthy status with one command (Q4)

**Scenario ID**: LEADERBOARD-INFRA-001.2-S1

**GIVEN**
* Docker and Docker Compose are installed on a clean machine
* The repository is checked out
* No database server is installed on the host

**WHEN**
* The developer runs `docker compose up`

**THEN**
* Both the `api` and `web` services report healthy status within 60 seconds
* A GET request to `http://localhost` returns HTTP 200 (SPA served by `web`)
* A GET request to the leaderboard endpoint returns HTTP 200 (API served by `api`)
* 0 database servers are installed or started on the host machine

---

### LEADERBOARD-INFRA-001.3: SQLite database on a Docker volume — data survives restart

**Architecture References**:
- `07-deployment-view.md` §"Local Development" ("SQLite is a file on a local Docker volume"); §"Production Environment" ("SQLite is a file on a named Docker volume"); §"Container Mapping" (Game Database row)
- `09-architecture-decisions.md` ADR-004 §"Decision" ("database file lives on a Docker volume in local development and production"); ADR-004 §"Consequences" ("Backups are file or volume backups")
- `08-cross-cutting-concepts.md` §"Persistence Access"

AS A developer
I WANT the SQLite database file stored on a named Docker volume that persists across container restarts
SO THAT submitted scores survive service restarts in both local development and production, as required by ADR-004

#### SCENARIO 1: Score written before restart is readable after restart

**Scenario ID**: LEADERBOARD-INFRA-001.3-S1

**GIVEN**
* The `api` service is running with the SQLite file on a named Docker volume
* A score row has been written to the database

**WHEN**
* The `api` container is stopped and restarted

**THEN**
* The previously written score row is present in the database after restart
* The SQLite file path matches the path configured via environment variable

#### SCENARIO 2: Tests use a temporary database, not the volume

**Scenario ID**: LEADERBOARD-INFRA-001.3-S2

**GIVEN**
* The pytest suite is running in CI (Q3 environment: no Game Client, no database server)

**WHEN**
* The test suite starts

**THEN**
* The Repository is configured with a temporary SQLite file path (not the production volume path)
* The temporary file is removed after the test run completes
* 0 database servers are installed or started

---

### LEADERBOARD-INFRA-001.4: Configuration via environment variables — no secrets in the repository

**Architecture References**:
- `08-cross-cutting-concepts.md` §"Configuration and Secrets" ("Non-secret settings such as the API URL, OIDC issuer and client ID, database path, and port come from environment variables. No secrets are stored in the repository; the detect-secrets pre-commit hook enforces this.")
- `09-architecture-decisions.md` ADR-003 §"Consequences" ("Changing provider would change token verification configuration, not the API contract")
- `02-constraints.md` §"Conventions" — pre-commit hooks mandatory

AS A developer
I WANT the Game API and API Client to read all non-secret settings (OIDC issuer, client ID, database path, port, API URL) from environment variables and for no secret values to be committed to the repository
SO THAT configuration is separated from code and the OIDC provider can be changed without altering the API contract, as required by Ch8 §"Configuration and Secrets"

#### SCENARIO 1: API starts correctly when required environment variables are set

**Scenario ID**: LEADERBOARD-INFRA-001.4-S1

**GIVEN**
* The `api` container is started with environment variables set for OIDC issuer, client ID, database path, and port

**WHEN**
* The `api` service starts

**THEN**
* The service reaches healthy status and responds to requests
* No hard-coded OIDC issuer, client ID, or database path is present in the application source files

#### SCENARIO 2: detect-secrets pre-commit hook blocks a committed secret

**Scenario ID**: LEADERBOARD-INFRA-001.4-S2

**GIVEN**
* A developer stages a file containing a string that detect-secrets classifies as a secret (for example an API key pattern)

**WHEN**
* The developer runs `git commit`

**THEN**
* The pre-commit hook exits with a non-zero code
* The commit is rejected and no secret is stored in the repository

---

### LEADERBOARD-INFRA-001.5: Request logging — one line per request, ID tokens and bodies never logged

**Architecture References**:
- `08-cross-cutting-concepts.md` §"Logging" ("The Game API logs one line per request with method, path, status code, and duration. ID tokens, secrets, and full request bodies are never logged.")
- `11-risks-and-technical-debts.md` R3 ("return a clear 401 so the client can prompt re-sign-in")

AS AN operator
I WANT the Game API to emit one structured log line per request containing method, path, status code, and duration, and to never log ID tokens, secrets, or full request bodies
SO THAT requests can be traced and diagnosed without leaking sensitive player data

#### SCENARIO 1: Successful score submission produces one log line with the correct fields

**Scenario ID**: LEADERBOARD-INFRA-001.5-S1

**GIVEN**
* The `api` service is running with logging enabled
* A valid score submission request is processed and returns HTTP 201

**WHEN**
* The log output is inspected

**THEN**
* Exactly one log line is emitted for that request
* The line contains: HTTP method (`POST`), the score submission path, status code `201`, and a numeric duration in ms
* The line does not contain the ID token value
* The line does not contain the full request body

#### SCENARIO 2: Rejected submission (401) is logged without the token

**Scenario ID**: LEADERBOARD-INFRA-001.5-S2

**GIVEN**
* A submission with no ID token is processed and returns HTTP 401

**WHEN**
* The log output is inspected

**THEN**
* Exactly one log line is emitted containing method `POST`, the submission path, status `401`, and a duration
* No token value or request body appears in the log

---

INFRA Events: N/A — the Leaderboard context uses synchronous REST/JSON over HTTPS for all interactions (`03-context-and-scope.md` §"Technical Context"; `09-architecture-decisions.md` ADR-001 §"Backend API"). No message queues or event topics are part of this architecture.

INFRA Backend tests (Q3): covered by LEADERBOARD-INFRA-001.1 — the CI pipeline runs pytest and pytest-bdd with a temporary SQLite database and no Game Client before publishing the image, satisfying Q3 ("0 browser processes; 0 configured database servers; all scenarios pass before the image is published").

---

## BE Sub-Stories

### LEADERBOARD-BE-001.1: Score Endpoint — accept a POST with score payload and return 201

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Score Endpoint ("Receives score submissions over REST/JSON"); §"Component View - Game API"
- `03-context-and-scope.md` §"Technical Context" (Score submission row: "REST/JSON over HTTPS, write request")
- `08-cross-cutting-concepts.md` §"API Contract and Versioning"; §"Error Handling"
- `09-architecture-decisions.md` ADR-001 §"Backend API"; ADR-004
- `10-quality-requirements.md` Q3, Q7

AS A developer
I WANT the Score Endpoint to accept a POST request with a JSON body containing a score, play time, and ID token and to return HTTP 201 on success
SO THAT the API Client has a tested, stable REST/JSON contract for score submission, satisfying Q7 and requirement 4

#### SCENARIO 1: Valid payload returns 201 (Q3)

**Scenario ID**: LEADERBOARD-BE-001.1-S1

**GIVEN**
* The pytest-bdd suite runs against a temporary SQLite database (no browser, no Game Client)
* A POST request is prepared with a JSON body `{"score": 100, "play_time_seconds": 30}` and a valid Authorization header

**WHEN**
* The request is sent to the Score Endpoint

**THEN**
* The response status code is `201`
* The response body is valid JSON

#### SCENARIO 2: Missing required field returns 422

**Scenario ID**: LEADERBOARD-BE-001.1-S2

**GIVEN**
* A POST request body is missing the `play_time_seconds` field

**WHEN**
* The request is sent to the Score Endpoint

**THEN**
* The response status code is `422`
* The response body contains a `detail` array describing the missing field (FastAPI standard 422 structure)
* 0 rows are written to the database

---

### LEADERBOARD-BE-001.2: Auth Verification — verify token signature, issuer, audience, and expiry (Q5)

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Auth Verification ("Verifies the OIDC ID token before a score is accepted")
- `08-cross-cutting-concepts.md` §"Authentication and Authorization" ("The API must check the token signature, issuer, audience, and expiry"); §"Error Handling" (401 for auth failures)
- `09-architecture-decisions.md` ADR-003 §"Consequences" ("The API must check the token signature, issuer, audience, and expiry")
- `10-quality-requirements.md` Q5 ("100% of unverified submissions rejected; 0 rows written; 0 passwords stored by the system")
- `11-risks-and-technical-debts.md` R3

AS A developer
I WANT Auth Verification to reject any submission whose ID token is missing, invalid, expired, issued by the wrong issuer, or intended for the wrong audience, returning HTTP 401 and writing 0 rows
SO THAT only verified players can submit scores, satisfying Q5 and requirement 5

#### SCENARIO 1: Valid ID token is accepted (Q3, Q5)

**Scenario ID**: LEADERBOARD-BE-001.2-S1

**GIVEN**
* The pytest-bdd suite runs with a fake identity provider that issues well-formed ID tokens
* A POST request carries an Authorization header with a valid, unexpired token signed by the fake provider, with the correct issuer and audience

**WHEN**
* Auth Verification processes the token

**THEN**
* Auth Verification passes control to Score Validation
* 0 rows are written at this stage by Auth Verification itself

#### SCENARIO 2: Missing token returns 401, 0 rows written (Q5)

**Scenario ID**: LEADERBOARD-BE-001.2-S2

**GIVEN**
* A POST request has no Authorization header

**WHEN**
* The Score Endpoint receives the request

**THEN**
* The response status code is `401`
* The response body contains a `detail` field
* 0 rows are written to the database

#### SCENARIO 3: Expired token returns 401, 0 rows written (Q5)

**Scenario ID**: LEADERBOARD-BE-001.2-S3

**GIVEN**
* The Authorization header contains a token whose expiry (`exp` claim) is in the past

**WHEN**
* Auth Verification processes the token

**THEN**
* The response status code is `401`
* 0 rows are written to the database

#### SCENARIO 4: Wrong audience returns 401, 0 rows written (Q5)

**Scenario ID**: LEADERBOARD-BE-001.2-S4

**GIVEN**
* The Authorization header contains a token with an `aud` claim that does not match the configured client ID

**WHEN**
* Auth Verification processes the token

**THEN**
* The response status code is `401`
* 0 rows are written to the database

#### SCENARIO 5: Bad signature returns 401, 0 rows written (Q5)

**Scenario ID**: LEADERBOARD-BE-001.2-S5

**GIVEN**
* The Authorization header contains a token whose signature has been tampered with

**WHEN**
* Auth Verification processes the token

**THEN**
* The response status code is `401`
* 0 rows are written to the database

---

### LEADERBOARD-BE-001.3: Score Validation — plausibility checks return 422, distinguishable from malformed request (Q6)

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Score Validation ("Applies plausibility checks to submitted scores")
- `08-cross-cutting-concepts.md` §"Input and Score Validation" ("Score Validation checks that a submitted score and play time are within plausible bounds before the Repository stores anything. This is a plausibility check, not anti-cheat.")
- `08-cross-cutting-concepts.md` §"Error Handling" ("422 shows a rejected score")
- `09-architecture-decisions.md` ADR-001 §"Consequences" ("the API can only run plausibility checks (e.g. maximum possible score, play time)")
- `10-quality-requirements.md` Q6 ("100% of implausible submissions rejected; 0 rows written")
- `11-risks-and-technical-debts.md` R1

> **GAP-001 applies here.** No numeric bounds for `MAX_SCORE` or `MIN_PLAY_TIME_SECONDS` are defined in the architecture. The scenarios use these named constants. Concrete values must be decided before this sub-story can be fully implemented.

AS A developer
I WANT Score Validation to reject submissions whose score exceeds `MAX_SCORE` or whose play time is below `MIN_PLAY_TIME_SECONDS`, returning HTTP 422 with a `detail` message that is distinguishable from a malformed-request 422
SO THAT implausible scores are rejected before storage and a client can tell a validation rejection from a schema error, satisfying Q6 and requirement 6

#### SCENARIO 1: Score within bounds passes validation (Q6)

**Scenario ID**: LEADERBOARD-BE-001.3-S1

**GIVEN**
* A valid ID token is present
* The request body has `score` ≤ `MAX_SCORE` and `play_time_seconds` ≥ `MIN_PLAY_TIME_SECONDS`

**WHEN**
* Score Validation runs

**THEN**
* Validation passes and control is handed to the Repository
* 0 rows are rejected at this stage by Score Validation

#### SCENARIO 2: Score above MAX_SCORE returns 422, 0 rows written (Q6)

**Scenario ID**: LEADERBOARD-BE-001.3-S2

**GIVEN**
* A valid ID token is present
* The request body has `score` > `MAX_SCORE`

**WHEN**
* Score Validation runs

**THEN**
* The response status code is `422`
* The response body contains a `detail` field that includes the word "score" and identifies it as a plausibility rejection (not a schema error)
* 0 rows are written to the database

#### SCENARIO 3: Play time below MIN_PLAY_TIME_SECONDS returns 422, 0 rows written (Q6)

**Scenario ID**: LEADERBOARD-BE-001.3-S3

**GIVEN**
* A valid ID token is present
* The request body has `play_time_seconds` < `MIN_PLAY_TIME_SECONDS`

**WHEN**
* Score Validation runs

**THEN**
* The response status code is `422`
* The response body `detail` field identifies the rejection as a plausibility failure and references `play_time_seconds`
* 0 rows are written to the database

---

### LEADERBOARD-BE-001.4: Repository — store Google subject ID and display name only (R8)

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — Repository ("Reads and writes scores in SQLite behind a simple interface"); §"Player Data Stored" ("stores only the Google subject ID and display name. Email address is not stored, and the ID token is verified and then discarded; it is never persisted.")
- `08-cross-cutting-concepts.md` §"Persistence Access"; §"Authentication and Authorization" ("Only the Google subject ID and display name are stored for each player; email is not stored and the ID token is never persisted.")
- `09-architecture-decisions.md` ADR-004 §"Decision"
- `11-risks-and-technical-debts.md` R8 ("Store only the Google subject ID and display name; never store email or the ID token")

AS A developer
I WANT the Repository to write exactly the Google subject ID, display name, score, and play time for each submission, and never write the player's email or the ID token
SO THAT player data is minimised and R8 is mitigated

#### SCENARIO 1: Repository writes the correct fields (Q3)

**Scenario ID**: LEADERBOARD-BE-001.4-S1

**GIVEN**
* The pytest suite runs with a temporary SQLite database
* A score submission has passed Auth Verification and Score Validation
* The verified token yields Google subject ID `"sub-123"` and display name `"Alice"`
* The submission has score `200` and play time `45` seconds

**WHEN**
* The Repository writes the submission

**THEN**
* Exactly one row is present in the database
* The row contains subject ID `"sub-123"`, display name `"Alice"`, score `200`, play time `45`
* The row does not contain an email field value
* The row does not contain the ID token

#### SCENARIO 2: Repository does not receive or store the ID token

**Scenario ID**: LEADERBOARD-BE-001.4-S2

**GIVEN**
* Auth Verification has verified the token and extracted subject ID and display name

**WHEN**
* The Repository write method is called

**THEN**
* The write method signature accepts subject ID, display name, score, and play time only — no token parameter
* The database schema has no column for email or ID token

---

### LEADERBOARD-BE-001.5: REST/JSON contract test — shape is verified in CI (Q7, R4)

**Architecture References**:
- `08-cross-cutting-concepts.md` §"API Contract and Versioning" ("The REST/JSON contract between the API Client and the two endpoints is explicit and covered by tests. FastAPI's generated API document is the reference for that contract.")
- `09-architecture-decisions.md` ADR-001 §"Consequences" ("The frontend and backend use two languages (TypeScript and Python), so the API contract must be kept explicit and tested.")
- `10-quality-requirements.md` Q7 ("0 undetected breaking changes; 100% of contract tests pass on the main branch")
- `11-risks-and-technical-debts.md` R4

AS A developer
I WANT a contract test that verifies the Score Endpoint request and response JSON shapes in CI, failing the pipeline if the shape changes without a version bump
SO THAT TypeScript and Python cannot silently drift apart, satisfying Q7 and mitigating R4

#### SCENARIO 1: Contract test passes when shape matches the API document (Q7)

**Scenario ID**: LEADERBOARD-BE-001.5-S1

**GIVEN**
* The GitHub Actions pipeline is running
* The Score Endpoint request schema is `{"score": integer, "play_time_seconds": number}` and the success response schema is `{"id": integer}` (or equivalent documented shape)

**WHEN**
* The contract test suite runs

**THEN**
* All contract tests pass with exit code 0
* The FastAPI-generated API document matches the tested schema

#### SCENARIO 2: A breaking shape change fails the pipeline before image publication (Q7)

**Scenario ID**: LEADERBOARD-BE-001.5-S2

**GIVEN**
* A developer renames a request field (for example `play_time_seconds` → `duration_seconds`) without updating the contract test

**WHEN**
* The pipeline runs

**THEN**
* The contract test step exits with a non-zero code
* No Docker image is published to GitHub Container Registry

---

## FE Sub-Stories

### LEADERBOARD-FE-001.1: API Client — sign in with Google and hold the ID token for the current session

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — API Client ("Submits the score with the ID token and reads the leaderboard; translates between contexts")
- `06-runtime-view.md` §"Log in with Google Sign-In" ("The API Client redirects the player to Google Sign-In using OAuth2/OIDC; after the player authenticates, the signed ID token is returned and kept by the API Client.")
- `08-cross-cutting-concepts.md` §"Authentication and Authorization" ("The API Client starts sign-in and holds the resulting ID token"); §"Configuration and Secrets" ("The SPA stores no client secret and keeps the ID token only for the current session.")
- `09-architecture-decisions.md` ADR-003

*See OQ-001: the sign-in mechanism (redirect vs. pop-up) is unresolved. The scenarios below are written neutrally.*

AS A player
I WANT to complete Google Sign-In from within the game and have the resulting ID token held by the API Client for the rest of the session
SO THAT I can submit my score without managing a password and without the token being stored beyond the current session

#### SCENARIO 1: Player completes Google Sign-In and the token is held in-session

**Scenario ID**: LEADERBOARD-FE-001.1-S1

**GIVEN**
* The player selects the sign-in option in the SPA
* The player completes Google Sign-In and returns to the SPA with a valid ID token (mechanism per OQ-001)

**WHEN**
* The API Client receives the ID token from the sign-in flow

**THEN**
* `APIClient.isSignedIn()` returns `true`
* The ID token is held in-memory for the current browser session only
* The ID token is not written to `localStorage`, `sessionStorage`, or any cookie
* The raw token value is not rendered anywhere in the UI

#### SCENARIO 2: Session ends — token is no longer available

**Scenario ID**: LEADERBOARD-FE-001.1-S2

**GIVEN**
* The player has previously signed in and the ID token is held in-memory

**WHEN**
* The browser tab is closed and a new session is opened

**THEN**
* `APIClient.isSignedIn()` returns `false`
* The player must sign in again to submit a score

---

### LEADERBOARD-FE-001.2: API Client — submit score at game over and map the response to a user message

**Architecture References**:
- `05-building-block-view.md` §"Building Block Responsibilities" — API Client; §"Component View - Game API"
- `06-runtime-view.md` §"Play a game and submit the score" ("At game over, Scoring hands the final score and play time to the API Client, which posts them with the ID token.")
- `08-cross-cutting-concepts.md` §"Error Handling" ("401 prompts a new sign-in on submission, 422 shows a rejected score, and network failures offer a retry. The client never displays raw stack traces or tokens.")
- `09-architecture-decisions.md` ADR-003; ADR-001 §"Consequences"
- `10-quality-requirements.md` Q5, Q6

AS A player
I WANT the API Client to POST my final score and play time with the ID token when my game ends, and to show me a clear message for every outcome — confirmation, re-sign-in prompt, rejection reason, or retry option — without displaying raw errors or token values
SO THAT I know whether my score was accepted and what to do if it was not

#### SCENARIO 1: Successful submission — confirmation shown

**Scenario ID**: LEADERBOARD-FE-001.2-S1

**GIVEN**
* The player has a valid ID token in-session
* The game has ended and a final score and play time are available

**WHEN**
* The API Client POSTs the score payload and receives HTTP 201

**THEN**
* A confirmation message is shown to the player (for example "Score submitted!")
* No raw HTTP status code, stack trace, or token value is displayed

#### SCENARIO 2: 401 response — player is prompted to sign in again

**Scenario ID**: LEADERBOARD-FE-001.2-S2

**GIVEN**
* The API Client POSTs the score payload and receives HTTP 401

**WHEN**
* The API Client processes the response

**THEN**
* The player sees a message prompting them to sign in again (for example "Please sign in to submit your score")
* No raw 401 status code, stack trace, or token value is displayed

#### SCENARIO 3: 422 response — rejection reason is shown

**Scenario ID**: LEADERBOARD-FE-001.2-S3

**GIVEN**
* The API Client POSTs the score payload and receives HTTP 422 with a `detail` message

**WHEN**
* The API Client processes the response

**THEN**
* The player sees the rejection reason from the `detail` field (for example "Score exceeds the maximum allowed value")
* No raw 422 status code or stack trace is displayed

#### SCENARIO 4: Network failure — retry option is offered

**Scenario ID**: LEADERBOARD-FE-001.2-S4

**GIVEN**
* The API Client attempts to POST the score payload and the network request fails (no response received)

**WHEN**
* The API Client processes the failure

**THEN**
* The player sees a message offering a retry option (for example "Submission failed. Try again?")
* No raw error object or stack trace is displayed

---

## Implementation Order

```
LEADERBOARD-INFRA-001.4   (env vars and secrets — no secrets in repo)
LEADERBOARD-INFRA-001.3   (SQLite volume — data store and test database)
LEADERBOARD-INFRA-001.1   (api image CI build, blocked on failing tests)
LEADERBOARD-INFRA-001.2   (one-command local start, Q4)
LEADERBOARD-INFRA-001.5   (request logging — one line, no tokens)
       ↓
LEADERBOARD-BE-001.1      (Score Endpoint — POST, 201, 422 on bad schema)
LEADERBOARD-BE-001.2      (Auth Verification — Q5: missing/expired/wrong
                           audience/bad signature all return 401, 0 rows)
LEADERBOARD-BE-001.3      (Score Validation — Q6: MAX_SCORE and
                           MIN_PLAY_TIME_SECONDS, 422 distinguishable
                           from schema error; blocked on GAP-001 resolution)
LEADERBOARD-BE-001.4      (Repository — subject ID + display name only, R8)
LEADERBOARD-BE-001.5      (Contract test — shape verified in CI, Q7)
       ↓
LEADERBOARD-FE-001.1      (API Client sign-in — token in-session only;
                           blocked on OQ-001 resolution)
LEADERBOARD-FE-001.2      (API Client submit — 201/401/422/network mapping)
       ↓
LEADERBOARD-STORY-001     (end-to-end: player plays a full game
                           [requires GAMEPLAY-STORY-005], signs in,
                           submits score, sees confirmation,
                           row in DB with subject ID + display name only)
```
