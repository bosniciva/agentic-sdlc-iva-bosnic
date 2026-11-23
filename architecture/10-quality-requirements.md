# 10. Quality Requirements

The quality tree below restates the five top quality goals from
[Chapter 1](./01-introduction-and-goals.md) as testable attributes. The scenarios
use stimulus, environment, response, and measure so they can be checked in CI or
in a manual smoke test.

## Quality Tree

- **Performance**
  - Rendering frame rate
  - No network in the gameplay loop
- **Testability**
  - Game logic isolated from canvas and browser
  - API testable without the frontend
- **Operability**
  - One-command local startup
  - No database server
- **Security**
  - Verified ID token on score submission
  - No locally stored passwords
  - Plausibility checks on submitted scores
- **Maintainability**
  - Explicit and tested REST/JSON API contract

## Quality Scenarios

| ID | Stimulus | Environment | Response | Measure |
|----|----------|-------------|----------|---------|
| Q1 | A player moves PacMan continuously for a full game. | Game Client in a desktop browser on the course workstation. | The Renderer draws each frame with no server round trip. | 60 FPS sustained; frame time within the ~16.7 ms budget; 0 network requests during play. |
| Q2 | A developer runs the game-logic unit tests. | Node 20 with Vitest; no browser, canvas, or API running. | Vitest tests for Game Engine, Maze, Ghost AI, and Scoring execute and pass. | 100% of game-logic tests run without a canvas or browser; 0 frontend or API processes required. |
| Q3 | A developer runs the pytest and pytest-bdd suite. | Local machine or GitHub Actions with a temporary SQLite database; no Game Client. | Score Endpoint, Auth Verification, Score Validation, and Repository scenarios pass. | 0 browser processes; 0 configured database servers; all scenarios pass before the image is published. |
| Q4 | A new developer starts the system for the first time. | Clean machine with Docker installed and the repository checked out. | The `api` and `web` services become available locally. | 1 command; 0 database servers installed; both services healthy. |
| Q5 | A score submission arrives with no ID token, or with an invalid or expired one. | Game API running with Google Sign-In configured. | Auth Verification rejects the request and no score is stored. | 100% of unverified submissions rejected; 0 rows written; 0 passwords stored by the system. |
| Q6 | A signed-in player submits a score above the maximum or with an impossible play time. | Game API with a valid ID token. | Score Validation rejects the submission and the client shows the rejection. | 100% of implausible submissions rejected; 0 rows written. |
| Q7 | A change alters a request or response shape without a version bump. | GitHub Actions running the contract tests. | The pipeline fails before the Docker image is published. | 0 undetected breaking changes; 100% of contract tests pass on the main branch. |
