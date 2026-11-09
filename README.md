## 🔬 Static Code Analysis (SonarQube)

### Research summary
Static code analysis inspects source code without running it, so bugs, security weaknesses and maintenance problems are found early. SonarQube Cloud is a hosted service that does this and shows the findings, ratings and a Quality Gate on a dashboard. A **bug** is code likely to behave incorrectly, a **vulnerability** is a weakness an attacker could exploit, and a **code smell** is code that works but is hard to maintain. A **Quality Gate** is a set of pass or fail conditions (for example, enough test coverage on new code) that decides whether a change is good enough. We run the scan from the CI pipeline and not only from the dashboard, so every push is analysed automatically and a failing gate stops the pipeline. Coverage is useful because it tells SonarQube which lines the unit tests actually executed. A GitHub Actions **secret** is encrypted and hidden in logs (used for the Sonar token), whereas a **variable** is plain text and visible.

### How it runs
SonarQube Cloud analyses the **backend** inside the GitHub Actions pipeline (`.github/workflows/ci.yml`, backend job). SonarQube's *Automatic Analysis* is switched off because the scan has to run in CI so that it can use the Jest-style coverage report.

1. Check out the full Git history (`fetch-depth: 0`)
2. Install dependencies (`npm ci`)
3. Run the unit tests with coverage (`npm run test:coverage`). `c8` wraps Node's test runner and writes `coverage/lcov.info`
4. Run the SonarQube Cloud scan with `sonar.qualitygate.wait=true`, so the job fails if the Quality Gate fails
5. Run the dependency audit

The Newman API tests depend on the backend job (`needs: backend`), so they only run if the tests **and the Quality Gate** pass.

The Sonar token, organisation key and project key are stored as GitHub Actions **secrets** (`SONAR_TOKEN`, `SONAR_ORGANIZATION`, `SONAR_PROJECT_KEY`) and are never committed. The analysis settings are in `hustlehub-backend/sonar-project.properties`:

| Property | Purpose |
|---|---|
| `sonar.sources=.` | Analyse the whole backend folder |
| `sonar.tests=test` and `sonar.test.inclusions` | Mark `test/**/*.test.js` as test code, not application code |
| `sonar.exclusions` | Skip `node_modules`, `coverage`, `ssl` and JSON files |
| `sonar.coverage.exclusions=test/**,server.js` | Test files and the start-up file are not counted for coverage |
| `sonar.javascript.lcov.reportPaths=coverage/lcov.info` | Where SonarQube reads the coverage report |

Only the backend is scanned (`projectBaseDir: hustlehub-backend`).

### Results
| Measure | Result | What it means |
|---|---|---|
| **Quality Gate** | **Passed** | New code meets the gate's conditions, including at least 80% coverage on new code |
| **Security** | A, 0 open issues | Vulnerabilities such as injection |
| **Reliability** | A, 0 open issues | Bugs: code likely to misbehave |
| **Maintainability** | A, 0 open issues | Code smells: code that is harder to maintain |
| **Coverage** | 77.3% (on 2.7k lines to cover) | Share of lines executed by the unit tests |
| **Duplications** | 1.7% (on 3.1k lines) | Share of repeated code |

Locally, `npm run test:coverage` runs **103 unit tests in 13 files (all passing)** and reports **80.8% statement coverage** for the files the tests load. SonarQube's overall figure is lower, because it also counts files that no unit test loads (the routes, `app.js` and `gigController.js`), which are only exercised by the Newman tests.

### Evidence
| Evidence | Screenshot |
|---|---|
| Pipeline with the coverage and SonarQube steps passing | <img width="500" height="281" alt="Pipeline passing with coverage and SonarQube steps" src="https://github.com/user-attachments/assets/de3f168a-8258-4b51-8387-e500be68fcc6" /> |
| Quality Gate failing: 0% coverage on new code | <img width="500" height="281" alt="Quality Gate failed on 0% coverage of new code" src="https://github.com/user-attachments/assets/a7a08f94-9adb-4cbf-b47e-da6d6137f53c" /> |
| Quality Gate passing after tests were added | <img width="500" height="281" alt="Quality Gate passed after tests were added" src="https://github.com/user-attachments/assets/55811739-8932-479a-a15f-f833c3852115" /> |
| SonarQube Summary dashboard | <img width="500" height="281" alt="SonarQube Summary dashboard, Overall Code" src="https://github.com/user-attachments/assets/f67b6998-e45a-4b22-a5f0-566af46297ba" /> |
| Data flow for the `role` finding | <img width="500" height="281" alt="Data flow for the role finding" src="https://github.com/user-attachments/assets/6c0700c2-02f7-49f3-8164-f2aa6b05d757" /> |
| Sonar showing the new tests covering the changed lines | <img width="500" height="281" alt="Covered code highlighted by SonarQube" src="https://github.com/user-attachments/assets/ec153787-8112-4bae-953f-dc83ede6a5c9" /> |
| Issue marked false positive with our comment | <img width="500" height="281" alt="Issue marked false positive with comment" src="https://github.com/user-attachments/assets/3cc10216-d472-4849-b4ff-66fde182ec08" /> |
| Coverage by folder | <img width="500" height="281" alt="Coverage by folder" src="https://github.com/user-attachments/assets/83cf1ff3-063a-4704-8bd0-a97453c2022c" /> |

### Issues found and how we handled them
The first automatic scan of the whole repository reported 34 issues. The CI scan covers only the backend and reported **9**. We did not change code just to make warnings disappear. For each one we opened the data flow, read the explanation and decided whether the problem was real.

| Finding | Decision | Reasoning |
|---|---|---|
| NoSQL injection, `adminController.getUsers` (`role`, `search`) | **Fixed** (defence in depth) | The route's `listUsersValidators` already restricted `role`, but SonarQube cannot see middleware and the controller had no check of its own. We added an allowlist for `role` and a string check for `search`. The remaining `search` flow, which is escaped with `escapeRegex`, was marked a false positive. |
| NoSQL injection, `authController` register and login (`email`) | **False positive** | `email` is validated as a string and a valid address (`isString`, `isEmail`, `normalizeEmail`) before the controller runs, and `sanitizeBody` strips `$` keys. |
| NoSQL injection, `bookingController` (3) and `gigController` (1) | **False positive** | Route ids are checked with `isMongoId`, and `req.user.id` comes from a JWT verified with `jwt.verify` and re-checked against the database. A client cannot supply operators or forge it. |
| Log injection, `errorMiddleware` | **Fixed** | The request method and URL were logged as supplied. We now strip CR/LF so a crafted URL cannot forge extra log lines. |
| Prefer `Number.NaN` over `NaN`, `incomeController` | **Fixed** | A correct, behaviour-neutral recommendation. |

Every false positive has a comment on the issue in SonarQube recording why it is safe. After triage there are **no open issues**.

### How the Quality Gate affected our work
The first push after our fixes **failed the Quality Gate with "0.0% Coverage on New Code" (required ≥ 80%)**, because the lines we had changed had no tests. The failure also stopped the Newman job from running. Instead of weakening the gate, we added unit tests for the code that changed and for other security-relevant code, and the next run passed.

### Tests added to improve coverage
We wrote tests for behaviour that matters for security, not to raise a percentage:

- `Admincontroller.test.js`: filter handling (valid, invalid and object-valued `role`; escaped and non-string `search`), no password hash in responses, an admin cannot change their own status or role, and deactivating a user bumps `tokenVersion`
- `Authmiddleware.test.js`: missing, invalid, revoked and deactivated tokens; the role is taken from the database, not the token
- `Rolemiddleware.test.js`, `Sanitize.test.js` (`$` operators and script tags stripped), `Errormiddleware.test.js` (error mapping, no internal details leaked, log-line forgery blocked) and `Incomecontroller.test.js` (query value handling)

Lowest-covered files that are still on our improvement list: `emailService.js` (about 37%), `adminController.js` (about 39%) and `twoFactorService.js` (about 52%). Coverage before: [first scan %]. Coverage after: [latest %].

### ESLint, unit tests, Newman and SonarQube: why all four
They are not duplicates, because each catches a different kind of problem.

| Check | What it does | What it catches |
|---|---|---|
| **ESLint** (frontend) | Reads the code without running it and applies style and correctness rules | Unused variables, unsafe patterns, style inconsistencies |
| **Unit tests** (`node:test`, Vitest) | Run small pieces of code in isolation and compare the result with what is expected | Logic bugs, such as a wrong tax calculation or a role check that lets the wrong user through |
| **Newman** | Runs the Postman collection against the running API and database | Integration problems: routing, status codes, authentication flow, data shared between requests |
| **SonarQube Cloud** | Analyses the source for vulnerabilities, bugs and smells, measures how much of the code the tests execute, and enforces a Quality Gate | Security patterns such as injection, duplicated code, untested code |

Lint finds style issues, unit tests find logic bugs, Newman finds integration problems, and SonarQube finds deeper security and quality issues while showing which code is untested. Running all four gives layered protection.

### Limitations
- Only the backend is analysed; the frontend is not scanned by SonarQube.
- Coverage is partial: several files (for example `gigController.js`, the routes and `emailService.js`) have little or no unit-test coverage and are verified mainly by Newman.
- Marking an issue as a false positive is a judgement we recorded in each comment. It should be revisited if the validators change.
- Secrets are only available to pushes and same-repository pull requests, so the scan does not run for pull requests from forks.
