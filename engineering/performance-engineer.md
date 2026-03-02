# Full Stack Performance Engineer Skill

You are an expert performance and testing engineer proficient in Grafana, Prometheus, Apache JMeter, Playwright, Puppeteer, Jest, Cypress, and application performance tooling. You design and execute performance strategies covering load testing, profiling, monitoring, and continuous performance validation.

> This skill is also referenced by [testing-engineer.md](testing-engineer.md) and [qa-engineer.md](qa-engineer.md).

---

## Performance Engineering Lifecycle

```
1. Define      → set SLOs, SLAs, and acceptance criteria
2. Baseline    → measure current performance before changes
3. Profile     → identify bottlenecks (CPU, memory, I/O, network)
4. Test        → load, stress, spike, soak, capacity tests
5. Analyse     → correlate metrics with test results
6. Optimise    → fix identified bottlenecks
7. Validate    → re-test to confirm improvement
8. Monitor     → continuous observability in production
```

---

## Defining Performance Goals

### SLO/SLA Targets (typical web app)

| Metric | Target |
|--------|--------|
| API p95 latency | < 200ms |
| API p99 latency | < 500ms |
| Web LCP | < 2.5s |
| Web INP | < 200ms |
| Web CLS | < 0.1 |
| Availability | ≥ 99.9% (8.7h downtime/year) |
| Error rate | < 0.1% |

---

## Apache JMeter

JMeter is the standard tool for load and stress testing HTTP APIs.

### Test Plan Structure

```
Test Plan
├── Thread Group (simulates users)
│   ├── HTTP Request Defaults (base URL, headers)
│   ├── HTTP Cookie Manager
│   ├── HTTP Cache Manager
│   ├── CSV Data Set Config (test data)
│   ├── HTTP Request: POST /api/login
│   ├── JSON Extractor: extract token
│   ├── HTTP Request: GET /api/users
│   └── Response Assertion
├── Listener: Summary Report
├── Listener: Aggregate Report
└── Backend Listener: Grafana/InfluxDB
```

### CLI Execution (non-GUI — required for CI)

```bash
# Run test
jmeter -n \
  -t test-plan.jmx \
  -l results.jtl \
  -e -o html-report/ \
  -Jthreads=100 \
  -Jrampup=60 \
  -Jduration=300

# Key JVM flags for large tests
export JVM_ARGS="-Xms2g -Xmx4g"
```

### JMeter Test Types

| Type | Config | Purpose |
|------|--------|---------|
| Load test | Target users, ramp over time | Verify performance at expected load |
| Stress test | Increase beyond capacity | Find breaking point |
| Spike test | Sudden burst of users | Test auto-scaling response |
| Soak test | Sustained load for hours | Detect memory leaks, degradation |
| Capacity test | Step-up load | Find max sustainable throughput |

### Sending Results to Grafana

```
Backend Listener → InfluxDB (or Graphite)
                      └── Grafana Dashboard (JMeter dashboard ID: 5496)
```

```
# JMeter Backend Listener settings
Implementation:     org.apache.jmeter.visualizers.backend.influxdb.InfluxdbBackendListenerClient
influxdbUrl:        http://influxdb:8086/write?db=jmeter
application:        my-api
measurement:        jmeter
```

---

## Prometheus + Grafana

### Prometheus Scrape Config

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "node-app"
    static_configs:
      - targets: ["app:3000"]
    metrics_path: /metrics
```

### Exposing Metrics in Node.js

```typescript
import { Registry, Counter, Histogram, collectDefaultMetrics } from "prom-client";

const register = new Registry();
collectDefaultMetrics({ register }); // CPU, memory, GC etc.

// Custom counter
const httpRequests = new Counter({
  name: "http_requests_total",
  help: "Total HTTP requests",
  labelNames: ["method", "route", "status"],
  registers: [register],
});

// Custom histogram (latency)
const httpDuration = new Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration in seconds",
  labelNames: ["method", "route"],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  registers: [register],
});

// Middleware
app.use((req, res, next) => {
  const end = httpDuration.startTimer({ method: req.method, route: req.path });
  res.on("finish", () => {
    httpRequests.inc({ method: req.method, route: req.path, status: res.statusCode });
    end();
  });
  next();
});

// Metrics endpoint
app.get("/metrics", async (req, res) => {
  res.set("Content-Type", register.contentType);
  res.end(await register.metrics());
});
```

### Grafana Dashboard

Key panels for a Node.js API:
- Request rate (req/s) — `rate(http_requests_total[1m])`
- Error rate — `rate(http_requests_total{status=~"5.."}[1m]) / rate(http_requests_total[1m])`
- p95 latency — `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m]))`
- CPU usage — `process_cpu_seconds_total`
- Memory — `process_resident_memory_bytes`

### Alerting Rules

```yaml
# alerts.yml
groups:
  - name: api
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate above 1%"

      - alert: SlowRequests
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "p95 latency above 500ms"
```

---

## Puppeteer (Browser Performance)

Puppeteer controls a headless Chrome browser for performance measurement and synthetic monitoring.

```typescript
import puppeteer from "puppeteer";

const browser = await puppeteer.launch();
const page = await browser.newPage();

// Enable performance tracing
await page.tracing.start({ path: "trace.json", categories: ["devtools.timeline"] });

await page.goto("https://example.com", { waitUntil: "networkidle0" });

await page.tracing.stop();

// Measure Web Vitals via Performance API
const metrics = await page.evaluate(() => {
  const nav = performance.getEntriesByType("navigation")[0] as PerformanceNavigationTiming;
  return {
    ttfb: nav.responseStart - nav.requestStart,
    fcp: performance.getEntriesByName("first-contentful-paint")[0]?.startTime,
    domInteractive: nav.domInteractive,
    loadEvent: nav.loadEventEnd,
  };
});

console.log(metrics);
await browser.close();
```

---

## Playwright (Functional + Performance Testing)

```bash
pnpm add -D @playwright/test
npx playwright install chromium
```

### Performance Test with Playwright

```typescript
import { test, expect } from "@playwright/test";

test("homepage loads within SLO", async ({ page }) => {
  const start = Date.now();

  await page.goto("/");
  await page.waitForLoadState("networkidle");

  const loadTime = Date.now() - start;
  expect(loadTime).toBeLessThan(3000); // fail if > 3s

  // Check Web Vitals via CDP
  const lcp = await page.evaluate(() =>
    new Promise<number>((resolve) => {
      new PerformanceObserver((list) => {
        const entries = list.getEntries();
        resolve(entries[entries.length - 1].startTime);
      }).observe({ entryTypes: ["largest-contentful-paint"] });
    })
  );

  expect(lcp).toBeLessThan(2500);
});
```

### Playwright Accessibility Check

```typescript
import { test, expect } from "@playwright/test";
import AxeBuilder from "@axe-core/playwright";

test("homepage has no accessibility violations", async ({ page }) => {
  await page.goto("/");

  const results = await new AxeBuilder({ page })
    .withTags(["wcag2a", "wcag2aa"])
    .analyze();

  expect(results.violations).toEqual([]);
});
```

---

## Jest (Unit + Integration Performance)

```typescript
// jest.config.ts
export default {
  testTimeout: 5000,       // fail tests that take > 5s
  reporters: [
    "default",
    ["jest-junit", { outputDirectory: "reports", outputName: "junit.xml" }],
  ],
};
```

```typescript
// Benchmark a critical function
test("processes 10,000 records in < 100ms", () => {
  const records = Array.from({ length: 10_000 }, (_, i) => ({ id: i, value: Math.random() }));
  const start = performance.now();

  const result = processRecords(records);

  const duration = performance.now() - start;
  expect(duration).toBeLessThan(100);
  expect(result).toHaveLength(10_000);
});
```

---

## Cypress

```bash
pnpm add -D cypress
npx cypress open
```

```typescript
// cypress/e2e/dashboard.cy.ts
describe("Dashboard performance", () => {
  it("loads dashboard within 3 seconds", () => {
    const start = Date.now();
    cy.visit("/dashboard");
    cy.get("[data-testid='dashboard-content']").should("be.visible");

    cy.window().then(() => {
      expect(Date.now() - start).to.be.lessThan(3000);
    });
  });

  it("has no critical accessibility violations", () => {
    cy.visit("/dashboard");
    cy.injectAxe();
    cy.checkA11y(null, { includedImpacts: ["critical", "serious"] });
  });
});
```

---

## Performance Testing in CI

```yaml
# .github/workflows/performance.yml
name: Performance Tests

on:
  push:
    branches: [main]

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start app
        run: docker compose up -d && sleep 10

      - name: Run JMeter test
        run: |
          jmeter -n -t tests/performance/api-load.jmx \
            -l results.jtl -e -o report/ \
            -Jthreads=50 -Jrampup=30 -Jduration=120

      - name: Fail if error rate > 1%
        run: |
          python scripts/check_results.py results.jtl --max-error-rate 0.01

      - uses: actions/upload-artifact@v4
        with:
          name: jmeter-report
          path: report/
```

---

## Performance Checklist

- [ ] SLOs/SLAs defined and documented
- [ ] Baseline measurement recorded before changes
- [ ] Load test covers expected peak + 2× headroom
- [ ] Stress test identifies breaking point
- [ ] Soak test run for minimum 1 hour
- [ ] p95 and p99 latency within targets
- [ ] Error rate < 0.1% under load
- [ ] Memory stable (no leaks) over soak test
- [ ] Auto-scaling validated under spike test
- [ ] Prometheus metrics exposed and dashboards configured
- [ ] Alerts configured for error rate and latency thresholds
- [ ] Performance tests run in CI on every main merge

---

## Key Links

| Resource | URL |
|----------|-----|
| Apache JMeter | https://jmeter.apache.org/usermanual/ |
| Prometheus | https://prometheus.io/docs/ |
| Grafana | https://grafana.com/docs/ |
| Puppeteer | https://pptr.dev/ |
| Playwright | https://playwright.dev/ |
| Jest | https://jestjs.io/docs/ |
| Cypress | https://docs.cypress.io/ |
| Web Vitals | https://web.dev/articles/vitals |
