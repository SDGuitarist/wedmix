# Test Coverage Analysis - WedMix

## Current State

**Test coverage: 0%.** The project has no source code, no test files, and no test infrastructure configured.

| Area | Status |
|------|--------|
| Source code (`src/`) | Does not exist |
| Test files | None |
| Test framework (Jest) | Referenced in `npm test` script, but not installed |
| Test configuration (`jest.config.js`) | Missing |
| TypeScript config (`tsconfig.json`) | Missing |
| CI/CD pipeline | Not configured |

The `package.json` test script runs `jest`, but `jest`, `ts-jest`, and `@types/jest` are not listed in `devDependencies`.

---

## Recommendations

### 1. Install and configure the test toolchain

Before any tests can be written, the foundational infrastructure must exist:

- **Install Jest and TypeScript support**:
  `npm install -D jest ts-jest @types/jest`
- **Create `jest.config.js`** with `ts-jest` preset, mapping `rootDir` to `src/` and test match patterns to `tests/**/*.test.ts`.
- **Create `tsconfig.json`** (the CLAUDE.md already contains a recommended configuration).

### 2. Prioritize testing by layer as source code is written

The planned architecture (from CLAUDE.md) defines routes, controllers, services, middleware, and utilities. Each layer has different testing needs:

#### a. Services (highest priority)

Files: `src/services/spotify.ts`, `src/services/gpt.ts`

These encapsulate all external API calls (Spotify, OpenAI). They are the riskiest code in the application because they depend on third-party systems that can change behavior, rate-limit, or fail.

**What to test:**
- Token refresh logic and OAuth flows against a mocked Spotify API
- Graceful handling of Spotify rate limits (429 responses) and retry/backoff
- GPT prompt construction - verify that user preferences produce correct prompt text
- GPT response parsing - the AI output is unstructured; test that the parser handles malformed, truncated, or unexpected responses
- Network failure scenarios (timeouts, DNS errors, 5xx responses)

**Mocking strategy:** Mock `spotify-web-api-node` methods and the OpenAI HTTP client. Never make real API calls in unit tests.

#### b. Controllers (high priority)

Files: `src/controllers/` (playlist generation, Spotify auth handling)

Controllers contain business logic that combines service calls with request validation and response formatting.

**What to test:**
- Input validation: missing fields, wrong types, out-of-range values on playlist generation requests
- That the correct services are called with the correct arguments
- Response format matches the documented contract (`{ success, data }` / `{ success, error }`)
- Edge cases: empty playlists, duplicate tracks, maximum track limits

#### c. Routes / Integration tests (high priority)

Files: `src/routes/spotify.ts`, `src/routes/playlist.ts`

**What to test:**
- HTTP method and path correctness (`GET /api/spotify/auth`, `POST /api/playlist/generate`, etc.)
- Status codes: 200 for success, 400 for bad input, 401 for unauthenticated, 500 for server errors
- End-to-end request/response cycles using `supertest` against the Express app with mocked services
- CORS headers are present and correct
- OAuth callback handling: valid code exchange, invalid/expired codes, CSRF state mismatch

**Additional dependency:** `npm install -D supertest @types/supertest`

#### d. Middleware (medium priority)

Files: `src/middleware/` (auth, error handling, logging)

**What to test:**
- Auth middleware rejects requests without valid tokens and passes through valid ones
- Error-handling middleware catches thrown errors and returns the standard error response format
- Rate limiting middleware (if implemented) correctly throttles excess requests

#### e. Utilities (lower priority, but easy wins)

Files: `src/utils/`

Pure utility functions are the easiest to test and provide the most reliable coverage.

**What to test:**
- Any data transformation or formatting helpers
- Validation helpers (email format, playlist name constraints, etc.)

### 3. Add integration/contract tests for external APIs

Unit tests with mocks verify internal logic but not whether the mocks match reality. Add a small suite of **contract tests** that:

- Validate the shape of actual Spotify API responses against the interfaces used in the code (can run against Spotify's sandbox or recorded fixtures).
- Ensure that if Spotify changes their API response format, tests fail before production does.

These can run on a less frequent schedule (nightly or pre-release) rather than on every commit.

### 4. Set up CI/CD to enforce coverage

- Add a GitHub Actions workflow (`.github/workflows/test.yml`) that runs `npm test` on every push and pull request.
- Configure Jest to collect coverage (`--coverage`) and set minimum thresholds:
  ```json
  "coverageThreshold": {
    "global": {
      "branches": 70,
      "functions": 80,
      "lines": 80,
      "statements": 80
    }
  }
  ```
- Fail the CI build if coverage drops below thresholds.

### 5. Specific risk areas to cover first

Once source code exists, these scenarios carry the most risk and should be tested earliest:

| Risk Area | Why It Matters |
|-----------|---------------|
| Spotify OAuth token refresh | Silent auth failures break the entire app |
| GPT response parsing | Unstructured AI output is inherently unpredictable |
| Playlist generation with edge-case inputs | Empty preferences, very long preference strings, special characters |
| Environment variable validation at startup | Missing `SPOTIFY_CLIENT_ID` should fail fast with a clear error, not crash mid-request |
| Error response consistency | Clients depend on a stable error format |

### 6. Consider adding linting as a pre-test gate

Install ESLint and run it before tests in CI. Type errors and lint violations caught before test execution save debugging time. This is not a substitute for tests, but it catches a class of issues (unused variables, type mismatches) automatically.

---

## Summary

The project is at zero coverage with zero infrastructure. The recommended sequence is:

1. Install Jest + ts-jest + supertest
2. Create jest.config.js and tsconfig.json
3. As source code is written, add tests in this priority order: services > controllers > routes (integration) > middleware > utilities
4. Set up GitHub Actions CI to run tests on every PR
5. Enforce coverage thresholds to prevent regression
