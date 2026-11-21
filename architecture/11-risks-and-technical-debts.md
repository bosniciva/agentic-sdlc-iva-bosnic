# 11. Risks and Technical Debts

## Risks

| ID | Risk | Probability | Impact | Mitigation |
|----|------|-------------|--------|------------|
| R1 | Cheating through a client-controlled score | High | Medium | Keep plausibility checks in Score Validation and treat the leaderboard as informal. Do not add anti-cheat; the limitation is recorded in ADR-001 and Chapter 8. |
| R2 | SQLite write concurrency | Medium | Medium | Keep all writes behind the Repository, watch for write errors in logs, and switch to PostgreSQL through a new Repository implementation if the kata grows. See ADR-004. |
| R3 | External identity provider outage or misconfiguration | Medium | High | Only score submission needs login, so gameplay and leaderboard reads still work. Verify OIDC configuration in CI and return a clear 401 so the client can prompt re-sign-in. See ADR-003. |
| R4 | TypeScript and Python API contract drifting apart | Medium | High | Keep the REST/JSON contract explicit, use FastAPI's generated document as the reference, and run contract tests in CI that block image publication. Version breaking changes. See Chapter 8 and scenario Q7. |
| R5 | Scope creep (multiplayer, extra mazes, anti-cheat) | High | Medium | Enforce the out-of-scope lists in Chapters 1 and 3 and review every change in a pull request before merge. |
| R6 | Frame-rate regression | Medium | High | Keep game logic separate from rendering, keep network calls out of the game loop, and check the 60 FPS / ~16.7 ms budget in scenario Q1. |
| R7 | Single VM failure in production | Low | High | Accept for the kata. Keep deployment as portable Docker Compose, back up the SQLite volume, and move to a managed platform only if availability needs increase. See Chapter 7. |
| R8 | Privacy concern from storing player identifiers | Low | Medium | Store only the Google subject ID and display name; never store email or the ID token, and keep Repository access limited to those fields. See Chapters 5 and 8. |

## Technical Debt

No accidental or unplanned debt is recorded for the MVP. The entries below are
intentional trade-offs from ADR Consequences; they become debt if their trigger
occurs.

| Item | Current state | Trigger to resolve | Note |
|------|---------------|--------------------|------|
| SQLite write limit | Accepted for the kata | Concurrent write failures or a second running instance | Repository keeps the change local to one component (ADR-004). |
| Single VM operations: manual TLS renewal, backups, and no automatic scaling | Accepted for the kata | Production availability becomes a real requirement | Compose configuration stays portable (Chapter 7). |
| Plausibility-only score validation | Accepted limitation | A requirement for competitive integrity | Resolving it needs server-side simulation, which would change the architecture (ADR-001). |
