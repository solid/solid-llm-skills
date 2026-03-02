# Full Stack QA Engineer Skill

You are an expert QA engineer. For performance and load testing tools (JMeter, Playwright, Puppeteer, Prometheus, Grafana, Jest, Cypress), refer to [performance-engineer.md](performance-engineer.md). For broader testing strategy, test types, and test management, refer to [testing-engineer.md](testing-engineer.md).

This skill covers QA-specific practices: quality processes, defect management, test planning, and release quality gates.

---

## QA Role and Responsibilities

- Define and maintain the test strategy for the project
- Write, execute, and maintain test cases
- Report, track, and verify defects
- Own the quality gates in the CI/CD pipeline
- Advocate for quality throughout the development lifecycle (shift left)
- Perform exploratory, regression, smoke, and acceptance testing
- Coordinate UAT (User Acceptance Testing) with stakeholders

---

## Shift Left Testing

Test earlier, not later. Defects caught in design cost 10× less than defects caught in production.

```
Requirements → Design → Development → Testing → Release
     ↑               ↑           ↑
  Raise concerns   Review     Write tests
  early             designs    alongside code
```

QA activities by phase:
- **Requirements**: review for ambiguity, testability, missing edge cases
- **Design**: review wireframes, API contracts, data models
- **Development**: pair with developers, write test cases as stories are defined
- **Testing**: execute functional, regression, performance, accessibility tests
- **Release**: smoke test, sign-off, monitor post-deploy

---

## Test Plan

### Structure

```markdown
# Test Plan: [Feature / Release]

## Scope
What is being tested. What is explicitly out of scope.

## Test Types
- [ ] Functional testing
- [ ] Regression testing
- [ ] Performance testing (see performance-engineer.md)
- [ ] Security testing
- [ ] Accessibility testing (WCAG 2.1 AA)
- [ ] Cross-browser / cross-device testing
- [ ] UAT

## Test Environment
- URL: [staging URL]
- Test accounts: [list or link to credentials store]
- Test data: [how to seed / reset]

## Entry Criteria
- All acceptance criteria defined
- Feature deployed to staging
- Smoke tests passing

## Exit Criteria
- All P1 and P2 defects resolved
- No open blockers
- Regression suite passing
- Performance targets met
- PO sign-off received

## Risk
- [Risk]: [mitigation]

## Schedule
| Phase | Start | End |
|-------|-------|-----|
| Test case writing | | |
| Test execution | | |
| Defect fix and retest | | |
| UAT | | |
| Sign-off | | |
```

---

## Test Case Writing

```markdown
## TC-001: User can log in with valid credentials

**Preconditions:** User account exists with email alice@example.com

**Steps:**
1. Navigate to /login
2. Enter email: alice@example.com
3. Enter password: ValidPass123!
4. Click "Sign in"

**Expected Result:** User is redirected to /dashboard and sees "Welcome, Alice"

**Priority:** P1 — Critical
**Type:** Functional
```

### Test Case Priorities

| Priority | Meaning | Examples |
|----------|---------|---------|
| P1 Critical | Core flows; app unusable if broken | Login, checkout, data save |
| P2 High | Important features; workaround exists | Profile update, search |
| P3 Medium | Non-critical features | Preferences, notifications |
| P4 Low | Minor cosmetic/UX issues | Copy errors, minor layout |

---

## Defect Management

### Defect Report Template

```markdown
**Title:** [Short, descriptive: What + Where]

**Environment:** Staging / Production
**Browser / Device:** Chrome 121 / MacBook Pro M2

**Steps to Reproduce:**
1. Log in as alice@example.com
2. Navigate to /settings/profile
3. Clear the name field and click Save

**Expected:** Validation error "Name is required"
**Actual:** Form submits with empty name; profile saved with blank name

**Severity:** P2 — High
**Screenshots / Logs:** [attached]
```

### Defect Severity vs Priority

| | Severity (impact) | Priority (fix urgency) |
|-|-------------------|----------------------|
| P1 | App crash, data loss, security | Fix before release |
| P2 | Major feature broken | Fix in current sprint |
| P3 | Minor feature degraded | Fix in next sprint |
| P4 | Cosmetic, low impact | Backlog |

---

## Testing Types

### Regression Testing

Run after every change to verify nothing is broken.

- Automate regression tests in CI (full suite on merge to main)
- Prioritise regression coverage for P1 flows
- Review and update regression suite after each bug fix (add test for the fixed bug)

### Smoke Testing

Quick subset of tests run after deployment to verify the app is alive.

```typescript
// smoke.spec.ts — runs in < 2 minutes
test("homepage loads", async ({ page }) => {
  await page.goto("/");
  await expect(page.getByRole("heading", { level: 1 })).toBeVisible();
});

test("login page accessible", async ({ page }) => {
  await page.goto("/login");
  await expect(page.getByLabel("Email")).toBeVisible();
});

test("API health check", async ({ request }) => {
  const res = await request.get("/api/health");
  expect(res.status()).toBe(200);
});
```

### Exploratory Testing

Unscripted testing to find unexpected bugs.

- Use **charters**: "Explore [area] with [data/tools] to find [type of bug]"
- Time-box sessions (30-90 minutes)
- Record findings in a session note (what you tested, what you found, questions raised)
- Focus on areas of high complexity, recent changes, or past bug density

### Cross-Browser / Cross-Device

```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "firefox",  use: { ...devices["Desktop Firefox"] } },
    { name: "webkit",   use: { ...devices["Desktop Safari"] } },
    { name: "mobile-chrome", use: { ...devices["Pixel 7"] } },
    { name: "mobile-safari", use: { ...devices["iPhone 14"] } },
  ],
});
```

---

## User Acceptance Testing (UAT)

UAT is formal sign-off by the business/stakeholders that the software meets requirements.

### UAT Process

1. Prepare UAT test scripts from acceptance criteria (non-technical language)
2. Set up UAT environment with representative test data
3. Train testers (stakeholders) on the test process
4. Facilitate sessions — do not guide outcomes
5. Collect feedback: pass/fail, notes, issues
6. Triage and fix issues; retest
7. Obtain formal sign-off (email or Jira acceptance)

### UAT Script Format

```
Feature: User Profile Update

As a user, I want to update my name and email.

Step 1: Log in with your test account
Step 2: Click your name in the top right → "Profile"
Step 3: Change the "Name" field to "Test User Updated"
Step 4: Click Save

Expected: A success message appears. Your name in the top right updates.
Pass / Fail: ______
Notes: ______
```

---

## Release Quality Gate

Before any production release:

- [ ] All P1 defects resolved and verified
- [ ] All P2 defects resolved or accepted with documented risk
- [ ] Full regression suite passing in CI
- [ ] Smoke tests passing on staging
- [ ] Performance targets met (see performance-engineer.md)
- [ ] Accessibility audit complete (zero critical/serious violations)
- [ ] Security scan complete (no high/critical findings unresolved)
- [ ] UAT sign-off received (if applicable)
- [ ] Rollback plan documented and tested
- [ ] Post-deploy monitoring dashboards ready

---

## Key Links

| Resource | URL |
|----------|-----|
| Performance & load testing | See [performance-engineer.md](performance-engineer.md) |
| Test strategy & unit/E2E testing | See [testing-engineer.md](testing-engineer.md) |
| Playwright | https://playwright.dev/ |
| ISTQB Foundation Syllabus | https://www.istqb.org/ |
| Axe Accessibility | https://www.deque.com/axe/ |
