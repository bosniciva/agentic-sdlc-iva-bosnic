# 12. Glossary

## Domain Terms

| Term | Definition |
|------|------------|
| PacMan | The browser maze game in which the player controls a character that eats pellets while avoiding ghosts. |
| Maze | The single fixed layout of walls and corridors that defines where PacMan and the ghosts can move. |
| Pellet | A small item placed throughout the maze that PacMan eats to gain points. |
| Power pellet | A larger pellet that temporarily allows PacMan to eat ghosts. |
| Ghost | One of four enemies whose movement is controlled by the Ghost AI component. |
| Life | One of the remaining chances PacMan has before the game ends. |
| Score | The number of points earned in one game, computed by Scoring from pickups and collisions. |
| Play time | The duration of a game, sent with a score and used in plausibility checks. |
| Submitted score | A score sent to the Game API together with play time and an ID token. |
| Leaderboard | The ranked list of submitted scores that any player can view without logging in. |

## Technical Terms

| Term | Definition |
|------|------------|
| ADR | Architecture Decision Record, a numbered note that captures a hard-to-reverse decision with Status, Context, Decision, Rationale, and Consequences. |
| arc42 | The template that fixes the twelve chapter files in `architecture/`. |
| C4 | The model used for context, container, component, and deployment diagrams in this documentation. |
| Container | A runnable building block packaged with its dependencies, here the Docker Compose services `api` and `web`. |
| Component | A building block inside a container, such as Game Engine or Score Endpoint. |
| PlantUML | The text format used to define diagrams as versioned `.puml` files. |
| SVG | The generated vector image referenced by chapters and produced from PlantUML by the pre-commit hook. |
| Pre-commit hook | A repository script that runs checks and generates SVGs before a commit is accepted. |
| Bounded context | An explicit boundary around a model and its language, here Gameplay, Leaderboard, and Identity. |
| Gameplay context | The bounded context inside the Game Client that owns maze, ghosts, and the in-game score. |
| Leaderboard context | The bounded context in the Game API and Game Database that owns submitted scores and ranking. |
| Identity context | The external bounded context owned by Google Sign-In that manages player accounts. |
| Anti-corruption layer | A component that translates between contexts, here the API Client between Gameplay and Leaderboard. |
| SPA | A single-page application, here the React 18 client that runs in one browser page. |
| Game Client | The React 18 and TypeScript container that runs the game loop and renders the SPA. |
| Game API | The Python 3.12 and FastAPI container that accepts scores and serves the leaderboard. |
| Game Database | The SQLite container that stores submitted scores and player identifiers on a Docker volume. |
| React 18 | The frontend framework used to build the Game Client. |
| TypeScript | The language in which the Game Client and its game logic are written. |
| Node 20 | The fixed runtime for frontend tooling and tests. |
| Vitest | The frontend test runner used for game-logic unit tests on Node 20. |
| HTML5 Canvas | The browser drawing surface used by the Renderer to draw the maze. |
| FastAPI | The Python 3.12 web framework used to build the Game API. |
| REST/JSON | The request/response style and data format used between the API Client and the endpoints. |
| HTTPS | The transport used by the browser to reach the client and by the client to reach the API. |
| OIDC | OpenID Connect, the identity layer on OAuth2 used by Google Sign-In. |
| ID token | The signed proof of player identity that the API Client obtains and Auth Verification checks. |
| Identity Provider | The external Google Sign-In system that authenticates players and issues ID tokens. |
| Auth Verification | The Game API component that verifies an ID token before a score is accepted. |
| Score Validation | The Game API component that applies plausibility checks to a submitted score. |
| Plausibility check | A check that score and play time are within possible bounds, not a full anti-cheat mechanism. |
| Repository layer | The single component through which all reads and writes of SQLite pass. |
| SQLite | The embedded database that stores submitted scores in a file. |
| Docker volume | The persistent directory that holds the SQLite database file. |
| Docker Compose | The tool used to start the `api` and `web` services with one command. |
| GitHub Actions | The CI/CD system that builds, tests, and publishes the Docker image. |
| GitHub Container Registry | The registry in the GitHub toolchain that stores the published Game API image. |
| pytest and pytest-bdd | The Python frameworks used for API tests and behaviour-driven scenarios. |
| Game Engine | The client component that advances game state on each frame and coordinates the game components. |
| Ghost AI | The client component that determines ghost movement and behaviour. |
| Scoring | The client component that tracks pickups and computes the final score. |
| Renderer | The client component that draws the current game state to the Canvas. |
| Input Handler | The client component that converts keyboard events into movement intents. |
| API Client | The client component that submits scores, reads the leaderboard, and translates between contexts. |
| Score Endpoint | The FastAPI route that receives score submissions. |
| Leaderboard Endpoint | The FastAPI route that returns ranked scores without requiring login. |
| FPS | Frames per second, the unit used for the 60 FPS rendering goal. |
| Frame budget | The approximately 16.7 ms available to draw one frame at 60 FPS. |
| Environment variable | A source of non-secret configuration such as the API URL, OIDC issuer, and database path. |
| Reverse proxy | Caddy in the `web` service, which terminates TLS for the domain, serves the SPA, and forwards API calls to `api`. |
| Quality tree | The hierarchy of quality attributes listed in Chapter 10. |
| Quality scenario | A measurable statement with stimulus, environment, response, and measure. |
| Technical debt | Deferred work or an accepted trade-off that becomes costly if its trigger occurs. |
