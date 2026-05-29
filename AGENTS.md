# AGENTS.md — DashGit Repository

## Project overview

DashGit is a browser-based dashboard for viewing and managing work items (issues, pull requests, branches, notifications) across GitHub and GitLab repositories. It runs entirely in the browser as static HTML/JS and is deployed on GitHub Pages.

The repository is a monorepo with three independent components:

| Component | Language | Purpose |
|-----------|----------|---------|
| `dashgit-web/` | JavaScript (ES6 modules) | Main browser application — UI, API adapters, OAuth/PAT authentication |
| `oauth-exchange/` | Node.js / Express | OAuth2 token exchange proxy — prevents exposing OAuth secrets in the browser |
| `dashgit-updater/` | Java / Maven | Dependabot update combiner — generates combined PR payloads (referenced in docs but may not be present in the current tree) |

There is **no `package.json` at the repository root**. Each component is managed independently.

---

## Repository layout

```
dashgit-ag/
├── dashgit-web/
│   ├── app/                    # Static front-end source code
│   │   ├── git/                # GitHub & GitLab REST/GraphQL API adapters
│   │   ├── login/              # Token management, encryption, OAuth config
│   │   ├── oauth/              # OAuth2 PKCE implementation
│   │   ├── core/               # Data model, caches (labels, notifications, statuses)
│   │   ├── assets/             # CSS, images, Font Awesome icons
│   │   ├── index.html          # Application entry point
│   │   ├── Config*.js          # Configuration controller and view
│   │   ├── Index*.js           # Main index controller
│   │   └── Wi*.js              # Work-item controllers and views
│   ├── test/                   # Unit test suite (Mocha + Chai + Sinon)
│   │   ├── package.json        # Test dependencies and scripts
│   │   ├── setup.js            # Test environment initialization
│   │   ├── input/              # Test fixture inputs
│   │   ├── expected/           # Expected output fixtures
│   │   └── actual/             # Generated actual output (git-ignored)
│   ├── package.json            # Front-end app metadata
│   └── OAUTH2.md               # OAuth2 documentation and PKCE flow details
├── oauth-exchange/
│   ├── server.js               # Express OAuth proxy (exposes /exchange endpoint)
│   ├── package.json
│   └── Dockerfile
├── .github/
│   ├── workflows/test.yml      # CI: runs unit tests on Node.js 18.x
│   └── copilot-instructions.md
├── sonar-project.properties    # SonarCloud configuration
└── README.md
```

---

## Running the application

The front end is **static HTML/JS with no build step**. Serve `dashgit-web/` from any local HTTP server:

```bash
# Example using Python
python -m http.server 8080 --directory dashgit-web/

# Example using Node.js http-server
npx http-server dashgit-web/ -p 8080
```

Then open `http://localhost:8080/app/index.html` in a browser.

The OAuth exchange proxy is only needed for OAuth2 flows with custom apps:

```bash
cd oauth-exchange
npm install
# Set required environment variables, then:
node server.js
```

---

## Running tests

Unit tests live in `dashgit-web/test/` and use **Mocha 11**, **Chai 6**, and **Sinon 21**.

```bash
cd dashgit-web/test
npm install
npm test          # Run tests
npm run report    # Run tests + generate Mochawesome HTML report
```

The `actual/` directory must exist before running tests — CI creates it with `mkdir actual`. Create it locally if it is missing.

Test reports and fixtures are published as CI artifacts (`test-ut-report-files`).

---

## CI/CD

The workflow `.github/workflows/test.yml` triggers on:
- Push to any branch except `dependabot/**` and `v*` tags
- Pull requests targeting `main`
- Manual dispatch (`workflow_dispatch`)

It runs the `test-ut` job on `ubuntu-latest` with Node.js 18.x. PRs from the same repo and non-dependabot branches skip the job to avoid duplicate runs.

---

## Key source areas

### API adapters (`dashgit-web/app/git/`)

- `GitHubApi.js` / `GitHubAdapter.js` — GitHub REST API calls and response mapping
- `GitHubGraphql.js` — GitHub GraphQL queries (branches, statuses)
- `GitLabApi.js` / `GitLabAdapter.js` / `GitLabGraphql.js` — GitLab equivalents
- `GitStoreApi.js` / `GitStoreAdapter.js` — manager-repository operations (follow-up, update payloads)
- `Log.js` — API call logging utilities

### Authentication (`dashgit-web/app/login/`, `dashgit-web/app/oauth/`, `dashgit-web/OAUTH2.md`)

- `Login.js` / `LoginController.js` — PAT and OAuth login flow
- `Tokens.js` — token storage in sessionStorage/localStorage
- `Encryption.js` — PAT encryption/decryption with user-supplied password
- `OAConfig.js` — OAuth app configuration
- `OAuthApi.js` — OAuth2 token requests
- `pkce.js` — PKCE verifier/challenge generation
- `dashgit-web/test/TestOALogin.js` — unit tests for the OAuth login flow

### Work-item display (`dashgit-web/app/Wi*.js`)

- `WiController.js` — main controller; orchestrates API calls and view updates
- `WiControllerFollowUp.js` / `WiControllerUpdate.js` — advanced feature controllers
- `WiServices.js` — data transformation between API responses and the UI model
- `WiView.js` / `WiViewHeaders.js` / `WiViewRender.js` — HTML rendering

### Core model and caches (`dashgit-web/app/core/`)

- `Model.js` — shared data model
- `Config.js` — configuration read/write (localStorage)
- `LabelsCache.js`, `NotifCache.js`, `StatusesCache.js`, `StatusIndex.js` — two-level API response caches
- `Surrogates.js` — deduplication logic for providers sharing the same authenticated user

### Configuration UI (`dashgit-web/app/Config*.js`)

- `ConfigController.js` — handles provider add/edit/delete
- `ConfigValidation.js` — validates configuration fields
- `ConfigView.js` — renders the Configure tab

---

## Code conventions

- **No build step**: the front end uses ES6 module `import` statements resolved at browser load time. Libraries are loaded from CDN (Octokit, GitBeaker, Bootstrap).
- **No TypeScript**: all source files are plain `.js`.
- **Testing pattern**: test files import modules directly using Node.js with `--experimental-vm-modules`. Sinon stubs replace API calls; Chai `assert` is used for assertions.
- **No framework**: vanilla JS DOM manipulation throughout. No React, Vue, or Angular.
- **localStorage / sessionStorage**: configuration is persisted in `localStorage`; OAuth tokens live only in `sessionStorage`.

---

## Important constraints

- Do not add a build system or bundler to `dashgit-web/` — the no-build design is intentional.
- The `dashgit-updater/` Java module is referenced in the README and `sonar-project.properties` but may not be present in the working tree. Do not assume it exists.
- Changes to API adapters should be validated against the corresponding test fixtures in `dashgit-web/test/input/` and `dashgit-web/test/expected/`.
- OAuth secrets must never appear in `dashgit-web/` source files. The `oauth-exchange/` proxy exists precisely to keep secrets server-side.
- The `dashgit/follow-up` branch in the manager repository must not be deleted — it stores persistent follow-up data.
