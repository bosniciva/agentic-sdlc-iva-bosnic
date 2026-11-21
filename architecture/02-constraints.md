# 2. Architecture Constraints

These are fixed inputs to the design. They cannot be changed by this
architecture; goals and scope are in
[Chapter 1](./01-introduction-and-goals.md) and are not repeated here.

## Technical Constraints

- **Python 3.12** is the backend runtime. No other language or runtime version
  may be chosen for the API.
- **React 18** is the frontend framework.
- **Node 20** is the fixed frontend toolchain runtime.
- **Vitest** is the frontend test runner for game logic.
- **Docker** is the packaging and local runtime format.
- **GitHub Actions** builds and tests the project. No other CI/CD system is
  used.
- **pytest and pytest-bdd** are the test frameworks for the backend. Both are
  mandatory; no alternative backend runner replaces them.

SQLite, TypeScript, and Google Sign-In are decisions recorded in ADR-001, not
constraints. They are fixed for this system because that ADR is accepted.

## Organizational Constraints

- The system is a **course kata** in the Agentic SDLC course.
- Work is **individual**: one participant owns the kata end to end.
- **All artifacts are code in Git**: documentation, diagrams, configuration,
  source, and tests are versioned in the repository.
- The **Git workflow** requires every change to go through issue, branch,
  commit, and pull request. No change is pushed directly to the main branch.

## Conventions

- **arc42** fixes the chapter structure and file names in `architecture/`.
- **C4 diagrams with PlantUML** fix the diagram notation; C4-PlantUML includes
  are used per level and SVGs are generated from `.puml` by the pre-commit
  hook, never written by hand.
- **ADRs** use `ADR-NNN` numbering with Status, Context, Decision, Rationale,
  and Consequences. ADRs are never deleted.
- **Commit message format** is exactly
  `#<issue> <type>(<scope>): <description>`, for example
  `#1 feat(auth): add login endpoint`. Allowed types are `feat`, `fix`,
  `docs`, `test`, `refactor`, `chore`, `ci`, `perf`, and `style`.
- **Pre-commit hooks** are mandatory and must pass: trailing whitespace and
  end-of-file checks, black, isort, ruff (line length 88, target py312),
  bandit, detect-secrets, and the PlantUML hook that renders the SVGs.
