# 4. Solution Strategy

This chapter shows how the architecture intends to meet the quality goals from
[Chapter 1](./01-introduction-and-goals.md). The underlying choices are already
decided and recorded in [ADR-001](./09-architecture-decisions.md); this chapter
references them instead of repeating them. Chapter 5 turns the strategy into
building blocks.

| Top quality goal (Chapter 1) | Strategy / top-level pattern | Supporting decision |
|------------------------------|------------------------------|---------------------|
| 1 - Real-time performance | Run the game loop and rendering in the browser on the Canvas. Keep every network call out of the frame path, so gameplay depends only on local computation. | ADR-002; ADR-001, "Game loop and rendering" row. |
| 2 - Testability | Keep game logic (maze, ghosts, collisions) in modules separate from Canvas rendering so it can be tested directly. Expose score and leaderboard behaviour through an API that is driven with pytest and pytest-bdd before the UI exists. | ADR-001, "Game logic ... must be testable on its own" in Consequences, and the "Testing" row. |
| 3 - Simplicity of operation | Keep the backend small: a REST API with a repository layer over SQLite, packaged as the `api` service and published to GitHub Container Registry; the `web` service serves the SPA. No database server is needed locally. | ADR-004; ADR-001, "Backend API", "Data storage", and "Packaging and CI/CD" rows. |
| 4 - Security of score data | Delegate login to Google Sign-In and verify the ID token in the API when a score is submitted. Apply plausibility checks because the score comes from a client the player controls; store no passwords. | ADR-003; ADR-001, "Authentication" row. |
| 5 - Maintainability of the API contract | Treat the REST/JSON contract between the TypeScript client and the Python API as an explicit, tested boundary. The repository layer keeps storage details behind the API rather than exposing them to the client. | ADR-001, "Frontend", "Backend API", and "Testing" rows. |

The strategy adds no new technology, container, or external system beyond
ADR-001 and the constraints in Chapter 2. Real-time gameplay and
request/response traffic stay separate: the browser owns the frame loop, while
the API handles only score submission and leaderboard reads.
