# Full Stack Testing Engineer Skill

You are an expert testing engineer. For performance testing tools (JMeter, Playwright, Puppeteer, Prometheus, Grafana, Jest, Cypress), refer to [performance-engineer.md](performance-engineer.md).

This skill covers the full testing strategy, test types, quality gates, and test management beyond performance tooling.

---

## Testing Strategy

### Testing Pyramid

```
        ▲
       /E2E\          ← few, slow, high confidence (Playwright, Cypress)
      /──────\
     / Integr. \      ← moderate, test boundaries (Supertest, pytest)
    /────────────\
   /   Unit Tests  \  ← many, fast, isolated (Jest, Vitest, pytest)
  /────────────────────\
```

- **70% unit** — pure functions, business logic, utilities
- **20% integration** — API endpoints, DB queries, service boundaries
- **10% E2E** — critical user journeys only

### What to Test at Each Level

| Level | What | What NOT |
|-------|------|---------|
| Unit | Pure functions, transformations, edge cases | UI rendering, DB calls |
| Integration | API contracts, DB interactions, auth flows | Exact UI layout |
| E2E | Critical happy paths, key user journeys | Every edge case |

---

## Test Quality Principles

1. **Tests should be deterministic** — same code, same result, every run
2. **Tests should be independent** — no shared mutable state between tests
3. **Tests should be fast** — unit < 10ms; integration < 1s; E2E < 30s per test
4. **Tests should be readable** — Arrange / Act / Assert (AAA) pattern
5. **Tests should fail for the right reason** — assert the specific behaviour, not implementation

### AAA Pattern

```typescript
test("calculates correct order total with discount", () => {
  // Arrange
  const items = [{ price: 100 }, { price: 50 }];
  const discountPercent = 10;

  // Act
  const total = calculateTotal(items, discountPercent);

  // Assert
  expect(total).toBe(135); // 150 - 10%
});
```

---

## Test Data Management

- Use factories / builders for test data, not hardcoded objects
- Reset DB state between integration tests (transactions, truncate, or in-memory DB)
- Never use production data in tests
- Use `faker.js` / `factory_boy` for realistic synthetic data

```typescript
// TypeScript: test factory with faker
import { faker } from "@faker-js/faker";

function createUser(overrides: Partial<User> = {}): User {
  return {
    id: faker.string.uuid(),
    name: faker.person.fullName(),
    email: faker.internet.email(),
    createdAt: faker.date.past(),
    ...overrides,
  };
}

// Usage
const user = createUser({ name: "Alice" });
const users = Array.from({ length: 10 }, () => createUser());
```

---

## Mocking

```typescript
// Jest: mock a module
jest.mock("../services/email", () => ({
  sendEmail: jest.fn().mockResolvedValue({ success: true }),
}));

// Mock a specific method
const sendEmail = jest.spyOn(emailService, "sendEmail").mockResolvedValue({ success: true });

// Assert it was called
expect(sendEmail).toHaveBeenCalledWith({
  to: "alice@example.com",
  subject: "Welcome!",
});

// Reset mocks between tests
beforeEach(() => jest.clearAllMocks());
```

---

## API Integration Testing

```typescript
// Node.js + Supertest
import request from "supertest";
import { app } from "../app";
import { db } from "../db";

beforeAll(async () => { await db.migrate.latest(); });
afterAll(async () => { await db.destroy(); });

beforeEach(async () => {
  await db("users").truncate();
});

test("POST /users creates a user", async () => {
  const res = await request(app)
    .post("/users")
    .send({ name: "Alice", email: "alice@example.com" })
    .expect(201);

  expect(res.body).toMatchObject({ name: "Alice", email: "alice@example.com" });
  expect(res.body.id).toBeDefined();
});
```

---

## E2E Testing with Playwright

See [performance-engineer.md](performance-engineer.md) for Playwright setup and configuration.

```typescript
// tests/auth.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Authentication", () => {
  test("user can sign up and log in", async ({ page }) => {
    // Sign up
    await page.goto("/signup");
    await page.fill('[name="name"]', "Alice Test");
    await page.fill('[name="email"]', "alice+test@example.com");
    await page.fill('[name="password"]', "SecurePass123!");
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL("/dashboard");

    // Log out
    await page.click('[aria-label="User menu"]');
    await page.click("text=Sign out");
    await expect(page).toHaveURL("/login");

    // Log back in
    await page.fill('[name="email"]', "alice+test@example.com");
    await page.fill('[name="password"]', "SecurePass123!");
    await page.click('button[type="submit"]');
    await expect(page).toHaveURL("/dashboard");
  });
});
```

### Page Object Model

```typescript
// tests/pages/LoginPage.ts
export class LoginPage {
  constructor(private page: Page) {}

  async goto() { await this.page.goto("/login"); }
  async fillEmail(email: string) { await this.page.fill('[name="email"]', email); }
  async fillPassword(pw: string) { await this.page.fill('[name="password"]', pw); }
  async submit() { await this.page.click('button[type="submit"]'); }

  async login(email: string, password: string) {
    await this.goto();
    await this.fillEmail(email);
    await this.fillPassword(password);
    await this.submit();
  }
}
```

---

## Test Coverage

```bash
# Jest coverage
pnpm jest --coverage

# Vitest coverage
pnpm vitest run --coverage
```

### Coverage Targets (guidelines, not rules)

| Codebase area | Target |
|---------------|--------|
| Business logic / domain | ≥ 90% |
| API handlers | ≥ 80% |
| Utilities | ≥ 85% |
| UI components | ≥ 70% |

**Coverage is a floor, not a ceiling.** 100% coverage with poor assertions is worse than 80% with meaningful tests.

---

## Quality Gates in CI

```yaml
# .github/workflows/test.yml
jobs:
  test:
    steps:
      - run: pnpm test --coverage --ci
      - run: pnpm test:e2e
      - name: Coverage gate
        run: |
          # Fail if overall coverage < 80%
          node -e "
            const cov = require('./coverage/coverage-summary.json');
            const pct = cov.total.lines.pct;
            if (pct < 80) { console.error('Coverage ' + pct + '% < 80%'); process.exit(1); }
          "
```

---

## Test Reporting

- **JUnit XML** for CI integration (`jest-junit`, `pytest-junit`)
- **HTML reports** for human review (Playwright built-in, Allure)
- **Trend tracking** for test duration and flakiness over time

---

## Testing Checklist

- [ ] Unit tests for all business logic functions
- [ ] Edge cases covered: empty inputs, max values, error states
- [ ] Integration tests for all API endpoints
- [ ] Auth/authorisation tested (can A access B's data? → must fail)
- [ ] E2E tests for critical journeys (signup, purchase, key workflows)
- [ ] Accessibility checks in E2E suite (axe-core)
- [ ] Tests run in CI on every PR
- [ ] Coverage gate enforced in CI
- [ ] Flaky tests tracked and fixed promptly
- [ ] Test data uses factories, not hardcoded values

---

## Key Links

| Resource | URL |
|----------|-----|
| Jest | https://jestjs.io/docs/ |
| Vitest | https://vitest.dev/ |
| Playwright | https://playwright.dev/ |
| Testing Library | https://testing-library.com/ |
| faker.js | https://fakerjs.dev/ |
| Supertest | https://github.com/ladjs/supertest |
| Performance tools | See [performance-engineer.md](performance-engineer.md) |
