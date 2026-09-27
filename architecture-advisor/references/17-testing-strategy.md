# 17 — Testing Strategy

## What this category covers

The balance between unit, integration, and end-to-end tests; test-driven development vs test-after workflows; contract testing between services; and test data management. This is also the concrete home for the vulnerability/dependency-scanning and CI-integrated security-testing mechanism that `16-security.md` explicitly defers to "this file's testing layers."

This file holds the full, project-agnostic knowledge base. The skill filters this down to the options and pros/cons relevant to a specific project at generation time — this file itself does not recommend a winner.

---

## Axis 1: Testing pyramid balance

### Unit-heavy (traditional testing pyramid: many unit tests, fewer integration, few e2e)

- **Pros:** fast to run and cheap to maintain, giving rapid feedback during development; failures pinpoint the exact broken unit precisely, making debugging fast; the full suite can run on every save or commit without meaningful CI cost
- **Cons:** heavy unit-test coverage with mocked dependencies can pass while the system as a whole is still broken — mocks can drift out of sync with the real behavior of what they're mocking, giving false confidence; doesn't catch integration-level bugs (how components actually work together), which is often where real bugs live

### Integration-heavy ("testing trophy": more integration tests, fewer unit tests, minimal e2e)

- **Pros:** tests closer to how the system is actually used — exercising real component boundaries and interactions catches the class of bug (integration mismatches) that heavily-mocked unit tests miss; often better return on testing investment per test written, since one integration test can cover what several narrow unit tests would need to separately verify
- **Cons:** slower to run than unit tests, and a failure doesn't pinpoint the exact broken line the way a unit test does, so diagnosing which component is at fault takes more investigation; more setup complexity per test — spinning up a real or realistic dependency, e.g. a test database

### E2E-heavy (many end-to-end tests exercising the full system through the UI/API)

- **Pros:** the highest fidelity — tests the system exactly as a real user or client would experience it, catching issues unit and integration tests structurally can't see (real deployment configuration, real cross-service wiring)
- **Cons:** the slowest and most brittle of the three — flaky due to timing, environment, and external dependencies; a failure gives the least specific information about what actually broke; expensive to maintain and run at scale, so heavy reliance on e2e as the primary testing layer doesn't scale well as the suite grows

**What belongs at each layer:** unit tests fit pure business logic, algorithms, and edge cases in isolated functions or classes — fast, deterministic, no I/O. Integration tests fit how components interact with real or realistic dependencies — database queries actually hitting a test database, API endpoints exercised with a real HTTP client, service-to-service calls. E2E tests fit critical user journeys end-to-end through the real or production-like system — login, checkout, the handful of flows where failure would be genuinely costly to the business.

Most real projects land on a hybrid rather than a pure pyramid, with a common and increasingly favored middle ground weighting toward integration tests as the bulk of the suite (the "trophy" shape), a solid unit-test base for pure logic, and a thin e2e layer for critical user journeys only. The right proportions genuinely differ by project — a system with substantial complex UI benefits from more e2e coverage of critical flows than a pure backend API service does.

**Terminology note:** "system testing" is this axis's e2e testing under a different, commonly-used name — testing the fully integrated system as a whole rather than individual components. No separate treatment needed; it's the same thing described above.

---

## Axis 2: Manual vs automated testing

### Manual testing

- **Pros:** no authoring or maintenance cost for automation infrastructure — a human simply uses the system directly; uniquely good at exploratory testing and catching "this feels wrong" issues a scripted assertion wouldn't think to check for, since a human can improvise and follow their own curiosity; no false sense of security from a passing suite that doesn't actually reflect real usage
- **Cons:** doesn't scale — the same test has to be re-run by a human every single time, which is slow and blocks fast iteration; inconsistent between runs and between testers, since human execution varies; can't realistically run on every commit the way automated tests can, so regressions are caught later and more expensively

### Automated testing

- **Pros:** repeatable and consistent — the exact same test runs the exact same way every time, removing human execution variance; scales to run on every commit or pull request without additional per-run human labor, catching regressions immediately rather than at the next manual pass; enables the CI/CD velocity `19-deployment-infra.md` and `20-environment-strategy.md` assume is possible
- **Cons:** real upfront authoring cost, and ongoing maintenance cost as the system changes — a test written against old behavior needs updating, not just the code being tested; can only verify what it was explicitly written to check, lacking a human's ability to notice something unexpected and off-script

Automated testing is the default backbone for anything beyond a small, early-stage project, for the reasons above — but manual testing remains genuinely valuable and isn't fully replaceable by automation, specifically for exploratory testing and usability evaluation (see "Non-functional testing types" below), where the point is discovering the unexpected rather than confirming the expected. Most mature projects run both: an automated suite as the safety net, with deliberate, periodic manual or exploratory sessions layered on top, rather than treating manual testing as something automation is meant to eventually eliminate entirely.

---

## Axis 3: Test-driven development vs test-after

### Test-driven development (write tests before implementation)

- **Pros:** forces thinking through the interface and requirements before writing code, often resulting in more testable, more decoupled design; the test suite grows in lockstep with the code, avoiding the common failure mode of tests being deprioritized after the fact
- **Cons:** real workflow discipline required, and can slow initial development velocity — especially for exploratory or prototype work where the right interface isn't yet known; done mechanically, can produce tests that over-fit to the first implementation's specific internals rather than testing genuine behavior

### Test-after (write the implementation first, tests afterward)

- **Pros:** faster initial development, especially useful for exploratory work where the right design isn't yet clear and would be premature to lock in via tests written first
- **Cons:** tests are more easily skipped or deprioritized under time pressure, since there's no forcing function requiring them before code is considered "done"; can result in tests that just confirm what the code already does — including any bugs — rather than testing intended behavior

This is genuinely a team-workflow preference rather than a strict architectural fork with a clear winner — most teams land somewhere in between (TDD for well-understood, logic-heavy code; test-after for exploratory or UI-heavy work) rather than dogmatically picking one for the whole codebase.

---

## Axis 4: Contract testing

Relevant specifically once the system involves multiple independently-deployable services with real API dependencies between them (`01-system-level.md`, `07-api-design.md`) — a conditional category, the same treatment `10-messaging.md` and `14-resilience.md` give their own conditional axes. For a monolith, this axis collapses to not-applicable.

### No contract testing (rely on integration/e2e tests alone to catch API mismatches)

- **Pros:** no additional tooling or discipline required beyond the testing layers in Axis 1
- **Cons:** in a microservices topology, a breaking API change in one service is only caught by integration or e2e tests that actually exercise the affected call path — easy to miss if cross-service test coverage isn't comprehensive, and even integration tests often run against a specific version of a dependency rather than catching drift over time

### Contract testing (e.g. consumer-driven contracts)

- **Pros:** explicitly verifies that a service's API still satisfies what its consumers expect, without needing a full integration test spinning up both services together — catches breaking changes early, at the point the producing service changes, rather than only when a consumer's own tests happen to catch it; scales better than full integration testing across many services, since each service tests against contracts rather than every real consumer
- **Cons:** requires deliberate tooling and process to define and maintain contracts, and discipline from every team involved; adds a category of test artifact (the contract itself) that needs to stay in sync with the API's actual evolution

---

## Axis 5: Test data management

### Shared/persistent test data

- **Pros:** simple to set up initially — one dataset used across test runs
- **Cons:** tests can interfere with each other if run in parallel, or if one test's side effects leak into another's expected starting state — leading to flaky, order-dependent tests that are hard to diagnose

### Isolated/ephemeral test data (fresh fixtures or seed data per test or test run)

- **Pros:** each test starts from a known, clean state, eliminating order-dependency and cross-test interference — the standard expectation for a reliable test suite; enables safe test parallelization, meaningfully speeding up CI as the suite grows
- **Cons:** needs deliberate setup/teardown machinery (fixtures, factories, database transactions rolled back per test), which is real infrastructure to build and maintain; can add per-test overhead that adds to suite runtime, needing to be balanced against the parallelization gains it enables

Isolated/ephemeral test data is the standard expectation for anything beyond a small, early-stage test suite — shared/persistent test data's flakiness tends to compound painfully as the suite grows, similar to how technical debt compounds elsewhere in this skill.

---

## Axis 6: Test prioritization & coverage strategy

### Coverage-target-driven (aim for a specific code-coverage percentage)

- **Pros:** a simple, objective, easy-to-communicate number to track and report on; ensures at least some baseline level of testing across the whole codebase rather than leaving large untested gaps by omission
- **Cons:** chasing a coverage percentage as the goal itself, rather than as a byproduct of testing what matters, produces a well-known failure mode — low-value tests written just to execute a line of code without meaningfully verifying its behavior, satisfying the metric without actually reducing risk; 100% coverage doesn't mean bug-free, and a lower-coverage suite that deliberately targets the riskiest code can catch more real bugs than a high-coverage suite that doesn't discriminate by risk

### Risk-based prioritization (test the highest-risk, highest-impact areas most thoroughly)

- **Pros:** concentrates testing effort where a bug would actually be most costly — payment logic, auth, data-loss-risk code paths — rather than spreading effort evenly regardless of consequence; a more efficient use of limited testing time and budget than treating all code as equally worth testing
- **Cons:** requires an actual risk assessment (which parts of the system are highest-risk) rather than a mechanical target, which is a judgment call that can be wrong or become outdated as the system changes; harder to communicate as a simple number the way a coverage percentage is, which can make it a harder sell to stakeholders wanting a simple metric

Risk-based prioritization is the more defensible default reasoning. Coverage percentage is still worth tracking as a useful diagnostic signal — a sudden drop is worth investigating — but shouldn't be the target being optimized for. The two aren't mutually exclusive; a healthy approach tracks coverage as a diagnostic while actually prioritizing test-writing effort by risk.

**Test suite evaluation:** worth a one-line note rather than a full option — coverage percentage measures how much code ran during tests, not whether the tests would actually catch a bug if one were introduced. Mutation testing (deliberately introducing small bugs into the code and checking whether the test suite catches them) is a more advanced technique for evaluating test *quality* rather than test *quantity*, worth flagging as an advanced/optional practice for a mature test suite rather than a default expectation — most projects can reasonably rely on periodic manual review of test suite health and flaky-test triage instead.

---

## Non-functional testing types

Beyond the functional-correctness testing covered in Axes 1-6 — collectively, what's sometimes called "functional testing": does the system do what it's supposed to do, as distinct from the non-functional qualities below — several other testing types matter for a production system but don't need their own full pros/cons axis. They're more a question of whether the project needs this kind of testing at all, which the upfront questions answer, than a structural trade-off with competing options.

- **Security testing (SAST/DAST, dependency scanning):** the concrete home for the vulnerability/dependency-scanning and CI-integrated security scanning `16-security.md` defers to this file. Static analysis (SAST) scans source code for known vulnerability patterns without running it; dynamic analysis (DAST) tests a running instance of the application for exploitable vulnerabilities; dependency scanning checks third-party packages against known-vulnerability databases. All three are typically wired into CI (`19-deployment-infra.md`'s pipeline) rather than run manually, and the real decision — per `16-security.md`'s vulnerability-management note — is scan frequency and whether a critical finding blocks a build or merely warns.
- **Performance/load testing:** verifies the system holds up under expected, or above-expected, traffic before it's a live production problem — deliberately generating load against a staging or dedicated test environment and measuring latency and error-rate degradation as load increases. Directly informs and validates the scaling decisions made in `05-scalability.md` rather than being a wholly separate concern — load testing is how a horizontal-scaling or caching strategy's assumptions actually get verified before real traffic tests them for the first time.
- **Usability testing:** evaluating ease of use, discoverability, and user satisfaction with real or representative users. Distinct from accessibility testing below (which is specifically about usability for users with disabilities or assistive-technology needs) and from the functional-correctness testing in Axes 1-6 (which verifies the system does what it's supposed to, but says nothing about whether it's pleasant or intuitive to use). Relevant for any project with a real UI (`12-frontend.md`); largely a manual, qualitative practice (Axis 2) rather than something automated testing substitutes for.
- **Accessibility testing:** relevant specifically for web/mobile projects with a UI (`12-frontend.md`) — automated accessibility scanning, which catches a meaningful subset of issues, plus manual or assistive-technology testing for what automation can't catch, for anything with real accessibility requirements or compliance obligations. Not relevant to a backend-only service or CLI tool.
- **Compatibility testing:** verifying the system works correctly across the range of browsers, devices, operating systems, and screen sizes real users will actually have — most relevant for web and mobile projects (`12-frontend.md`) with a diverse user base. A backend-only service or an internal tool with a controlled, known client environment has comparatively little need for this.
- **Installation testing:** verifying the install or setup process itself works correctly — relevant specifically for installable software (desktop apps, CLI tools, self-hosted deployments) rather than a centrally-hosted SaaS web app, where there's no separate "install" step for the end user to get wrong. Ties to `19-deployment-infra.md`'s compute-model decision for self-hosted deployment scenarios specifically.
- **Interrupt/interruption testing:** verifying the system behaves correctly when interrupted mid-operation — a phone call or notification interrupting a mobile app, a network connection dropping mid-request, the app being backgrounded and resumed. Most relevant for mobile (`12-frontend.md`/platform), and a good way to verify that `14-resilience.md`'s timeout, retry, and circuit-breaker patterns actually behave as designed under a real interruption rather than only in theory.
- **Acceptance testing (UAT):** distinct from the developer-driven testing layers above in *who* validates and *against what* — acceptance testing checks the system against business or user acceptance criteria, often performed by stakeholders or end users rather than developers, typically in a staging or UAT environment (`20-environment-strategy.md`). Worth naming explicitly as a practice a project can adopt — defining acceptance criteria upfront, often as Given/When/Then-style scenarios, and validating against them before release — rather than a technical testing layer with its own pros/cons table.

---

## What drives the recommendation (from upfront questions)

- **System-level shape** (from `01-system-level.md`): the direct driver for Axis 4 — a monolith has no cross-service API to contract-test; a microservices topology strengthens the case for contract testing in direct proportion to how many services have real dependencies on each other's APIs
- **Domain coupling / logic complexity**: complex business logic with many edge cases benefits more from a solid unit-test base, and possibly TDD (Axis 3), than a system that's mostly thin CRUD wrapping a database with comparatively little logic to verify
- **Platform**: web/mobile UI-heavy projects have more reason to invest in e2e coverage of critical flows, usability testing, accessibility testing, and compatibility testing than a backend-only API service does; mobile specifically raises the case for interrupt/interruption testing
- **Ops/deployment capacity**: low ops maturity weights toward relying on CI-integrated dependency scanning (already a low-cost default) over building out full SAST/DAST tooling, and toward a pragmatic testing-trophy mix over an ambitious, high-maintenance e2e suite
- **Time-to-ship pressure**: high pressure → a pragmatic trophy-leaning mix (solid unit tests for core logic, integration tests for the main paths, a thin e2e layer for the most critical flows only), test-after rather than strict TDD, coverage tracked as a diagnostic without chasing a target, deferring contract testing and dedicated performance testing until the system's shape or scale actually demands them, is a legitimate first version
- **Consistency requirements**: correctness-critical logic (financial calculations, anything with real-world irreversible consequences) pushes toward heavier unit-test coverage and stricter discipline for that specific logic (Axis 6's risk-based prioritization), even if the rest of the system takes a lighter approach overall

---

## Recommendation table skeleton

| Category | Option A | Option B | Option C | Recommendation + why | Approve |
|---|---|---|---|---|---|
| Testing pyramid balance | *Unit-heavy pros/cons* | *Integration-heavy (trophy) pros/cons* | *E2E-heavy pros/cons* | *most projects land on a trophy-shaped hybrid; justify against system complexity and UI-heaviness* | ☐ |
| Manual vs automated | *Manual pros/cons* | *Automated pros/cons* | — | *automated is the default backbone; manual remains valuable for exploratory/usability testing specifically, not a full substitute either way* | ☐ |
| TDD vs test-after | *TDD pros/cons* | *Test-after pros/cons* | — | *workflow preference, not a strict architectural fork; justify against logic complexity and exploratory-vs-well-understood work* | ☐ |
| Contract testing | *No contract testing pros/cons* | *Contract testing pros/cons* | — | *only relevant with multiple services (`01-system-level.md`); collapses to not-applicable for a monolith* | ☐ |
| Test data management | *Shared/persistent pros/cons* | *Isolated/ephemeral pros/cons* | — | *isolated/ephemeral is the standard default; justify against suite size and CI parallelization needs* | ☐ |
| Test prioritization & coverage | *Coverage-target-driven pros/cons* | *Risk-based prioritization pros/cons* | — | *risk-based is the more defensible default; track coverage as a diagnostic, not the optimization target* | ☐ |

Note: for most projects, especially early-stage ones, this can reasonably collapse to three rows ("trophy-shaped mix of unit and integration tests, thin e2e for critical flows, automated with test-after" and "isolated/ephemeral test data" and "risk-based prioritization, coverage tracked but not targeted") rather than six — drop the contract-testing row entirely for a monolith per its own guidance above.

---

## Related files

- `01-system-level.md` — contract testing's relevance (Axis 4) is driven directly by system-level shape
- `05-scalability.md` — performance/load testing validates the scaling assumptions made there
- `07-api-design.md` — contract testing verifies the API contracts designed there actually hold as services evolve
- `12-frontend.md` — usability, accessibility, and compatibility testing are all relevant specifically for UI-having projects covered there
- `14-resilience.md` — interrupt/interruption testing verifies that file's timeout/retry/circuit-breaker patterns actually behave as designed
- `16-security.md` — this file is the concrete home for that file's vulnerability/dependency-scanning and SAST/DAST deferral
- `19-deployment-infra.md` — the CI pipeline is where all of this file's testing layers actually get wired in and run automatically; installation testing ties to self-hosted deployment scenarios decided there
- `20-environment-strategy.md` — acceptance testing (UAT) typically happens in the staging/UAT environment decided there
