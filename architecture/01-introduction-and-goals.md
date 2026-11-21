# 1. Introduction and Goals

## Purpose

The PacMan system is a browser game that lets a player play a PacMan-style
maze game, submit a score, and see how that score ranks against other players
on a leaderboard. Gameplay runs entirely in the player's browser; the server
handles only scores and leaderboard requests.

The system is built as a coding kata in the Agentic SDLC course. Its purpose
is twofold: deliver a working game, and provide a small, realistic system on
which the course's architecture, requirements, CI/CD, and TDD/BDD agents can
be exercised.

The technology stack is fixed by ADR-001 in
[Chapter 9](./09-architecture-decisions.md): a React 18 / TypeScript
single-page application drawing on an HTML5 Canvas, a Python 3.12 / FastAPI
REST API, SQLite behind a repository layer, Google Sign-In (OIDC) for
authentication, and pytest / pytest-bdd for backend-first testing.

## Key Requirements

The following are the driving functional requirements. They are derived from
the system boundary shown in the [System Context diagram](./diagrams/c4-context.svg).

1. **Play the game.** A player can start and play a PacMan-style maze game in
   the browser: move PacMan, collect pellets, avoid or eat ghosts, and reach an
   end-of-game state with a final score. Playing the game does not require
   login.
2. **Real-time gameplay.** The game loop and rendering run client-side at a
   steady frame rate without network round trips.
3. **View the leaderboard.** Any player can view a ranked list of submitted
   scores without logging in.
4. **Submit a score.** At the end of a game, a player signs in and submits the
   resulting score to the API. Score submission is the only action that
   requires login.
5. **Authenticate players.** Login is delegated to an external OIDC identity
   provider (Google Sign-In). The API verifies the ID token sent with a score
   submission and does not store passwords.
6. **Validate submissions.** Because the score comes from a client the player
   controls, the API applies plausibility checks (for example, maximum
   possible score and play time). Full cheat prevention is out of scope.
7. **Testable backend first.** Score and leaderboard behaviour can be developed
   with pytest and pytest-bdd independently of the browser client. Game logic
   (maze, ghosts, collisions) is testable on its own, separate from rendering.
8. **Simple to run and deploy.** Docker Compose defines two services, `api`
   (FastAPI) and `web` (static server and reverse proxy serving the built SPA
   and forwarding API calls). Both are built and tested by GitHub Actions;
   local setup needs no database server.

## Out of scope

- One fixed maze and four ghosts only: no level editor, extra mazes, or
  additional enemy types.
- No multiplayer: one player per game session; the leaderboard is the only
  shared element.
- No full cheat prevention: client-supplied scores can only be plausibility
  checked by the API.

## Top Quality Goals

These goals are the main drivers for architectural decisions. They are stated
so they can be verified; the measurable quality scenarios are detailed in
[Chapter 10](./10-quality-requirements.md).

| Priority | Quality goal | Concrete target |
|----------|--------------|-----------------|
| 1 | Real-time performance | The client renders at 60 FPS; a frame fits within a ~16.7 ms budget and gameplay needs no server round trip. |
| 2 | Testability | Game logic is unit-testable without a canvas or browser; API behaviour is testable with pytest/pytest-bdd without running the frontend. |
| 3 | Simplicity of operation | A new developer can start the `api` and `web` Docker Compose services locally with one command; no external database server is required. |
| 4 | Security of score data | No passwords are stored; every score submission requires a verified OIDC ID token and passes plausibility checks. |
| 5 | Maintainability of the API contract | The TypeScript client and Python API share one explicit, tested REST/JSON contract so changes do not silently break either side. |

## Stakeholders

| Stakeholder | Expectation |
|-------------|-------------|
| Player | Plays a responsive game and views the leaderboard without logging in; signs in only to submit a score and does not manage a local password. |
| Course instructor / assessor | Sees the arc42 documentation, C4 diagrams, ADRs, and CI pipeline produced by the architecture, requirements, CI/CD, and TDD/BDD agents; can evaluate the kata end to end. |
| Developer (course participant) | Can understand the system from the documentation, run it locally with minimal setup, and extend it test-first. |
| Identity provider (external) | Owns user accounts and authenticates players; is treated as an external system, not part of the system under development. |
