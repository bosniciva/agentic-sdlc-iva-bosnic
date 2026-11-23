# 8. Cross-cutting Concepts

These concepts apply across the containers and components described in Chapters
5 to 7. Each one names the building blocks it affects.

## Authentication and Authorization

Google Sign-In is the only login method. The API Client starts sign-in and
holds the resulting ID token; the Auth Verification component in the Game API
verifies that token before a score is accepted. Authorization is minimal: any
valid ID token permits a score submission for that player, and there are no
roles or per-player permissions. The Leaderboard Endpoint needs no token. Only
the Google subject ID and display name are stored for each player; email is not
stored and the ID token is never persisted.

**Affects:** API Client, Score Endpoint, Auth Verification, Leaderboard
Endpoint, Identity Provider (external).

## Error Handling

API errors use FastAPI's JSON error body with a `detail` message; request
validation failures use the standard 422 detail structure, authentication
failures use 401, and unexpected failures use 500 with a generic message. The
API Client maps status codes to user-visible messages: 401 prompts a new
sign-in on submission, 422 shows a rejected score, and network failures offer a
retry. The client never displays raw stack traces or tokens.

**Affects:** Score Endpoint, Leaderboard Endpoint, Auth Verification, Score
Validation, API Client, Renderer.

## Logging

The Game API logs one line per request with method, path, status code, and
duration. ID tokens, secrets, and full request bodies are never logged. During
play the Game Client writes only errors to the browser console, so logging does
not consume the frame budget.

**Affects:** Game API components, API Client.

## Input and Score Validation

The Input Handler converts only supported keys into movement intents and
ignores all other input; the Game Engine and Maze reject moves that the layout
does not allow. Score Validation checks that a submitted score and play time
are within plausible bounds before the Repository stores anything. This is a
plausibility check, not anti-cheat: the client still controls the score.

**Affects:** Input Handler, Game Engine, Maze, Scoring, Score Validation.

## API Contract and Versioning

The REST/JSON contract between the API Client and the two endpoints is explicit
and covered by tests. FastAPI's generated API document is the reference for
that contract. Additive changes keep the same contract version; breaking
changes introduce a new major version so old clients are not silently broken.

**Affects:** API Client, Score Endpoint, Leaderboard Endpoint.

## Testing Approach

The API is developed test-first with pytest and pytest-bdd, covering endpoints,
auth verification against a fake identity provider, validation, and the
repository against a temporary SQLite database. Game logic is covered by Vitest
unit tests for the Game Engine, Maze, Ghost AI, and Scoring, with no canvas or
browser required. The Renderer and Input Handler get only thin integration
checks.

**Affects:** Game API components, Game Engine, Maze, Ghost AI, Scoring,
Renderer, Input Handler.

## Configuration and Secrets

Non-secret settings such as the API URL, OIDC issuer and client ID, database
path, and port come from environment variables. No secrets are stored in the
repository; the detect-secrets pre-commit hook enforces this. The SPA stores no
client secret and keeps the ID token only for the current session.

**Affects:** Game API, API Client, deployment configuration.

## Persistence Access

Only the Repository reads and writes SQLite; endpoints never construct SQL
themselves. This keeps the storage detail behind a small interface, makes tests
easy with a temporary database, and preserves the option to change the database
later without changing the API contract.

**Affects:** Repository, Score Endpoint, Leaderboard Endpoint, Game Database.
