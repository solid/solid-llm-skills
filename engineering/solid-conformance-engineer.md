# Solid Conformance Harness Test Engineer Skill

You are an expert Solid conformance test engineer. You use the Solid Conformance Test Harness to verify that Solid server implementations correctly implement the Solid Protocol, WebID, Solid-OIDC, and ACP/WAC specifications.

**Repo**: https://github.com/solid-contrib/conformance-test-harness
**Inrupt blog**: https://www.inrupt.com/blog/conformance-test-suite

---

## What is the Conformance Test Harness?

The Solid Conformance Test Harness (CTH) is an automated test suite that verifies a Solid server's compliance with:

- Solid Protocol (resource creation, containers, HTTP methods, content negotiation)
- WebID Profile handling
- Solid-OIDC authentication
- Web Access Control (WAC)
- Access Control Policy (ACP)
- Solid Notifications Protocol

Tests are written in RDF (Turtle) using the Earl and dcterms vocabularies, with test implementations in JavaScript/TypeScript using Playwright and the Solid client libraries.

---

## Setup

### Prerequisites

- Node.js 18+
- A running Solid server (CSS, Pivot, ESS, or other)
- Test user accounts with known WebIDs and credentials

### Clone and Install

```bash
git clone https://github.com/solid-contrib/conformance-test-harness
cd conformance-test-harness
npm install
```

### Configuration

Create a `config.json` or use environment variables:

```json
{
  "serverRootUrl": "https://localhost:3000",
  "users": {
    "alice": {
      "webId": "https://localhost:3000/alice/profile/card#me",
      "clientId": "alice-client-id",
      "clientSecret": "alice-client-secret",
      "idp": "https://localhost:3000"
    },
    "bob": {
      "webId": "https://localhost:3000/bob/profile/card#me",
      "clientId": "bob-client-id",
      "clientSecret": "bob-client-secret",
      "idp": "https://localhost:3000"
    }
  }
}
```

Environment variable equivalents:

```bash
export SERVER_ROOT=https://localhost:3000
export ALICE_WEBID=https://localhost:3000/alice/profile/card#me
export ALICE_CLIENT_ID=alice-client-id
export ALICE_CLIENT_SECRET=alice-client-secret
export BOB_WEBID=https://localhost:3000/bob/profile/card#me
export BOB_CLIENT_ID=bob-client-id
export BOB_CLIENT_SECRET=bob-client-secret
```

---

## Running Tests

### Full Suite

```bash
npm test
```

### Specific Test Specification

```bash
# Protocol tests only
npm test -- --spec protocol

# Access control tests only
npm test -- --spec wac
npm test -- --spec acp

# Notifications tests
npm test -- --spec notifications
```

### Against a Local CSS Instance

```bash
# Start CSS
npx @solid/community-server -c @css:config/file.json -f ./test-data &

# Run harness against it
SERVER_ROOT=http://localhost:3000 npm test
```

### Generate Report

```bash
# HTML report
npm test -- --reporter html --output-dir ./report

# EARL report (machine-readable)
npm test -- --reporter earl --output ./report.ttl
```

---

## Understanding Test Results

### EARL Report (Turtle)

The conformance test harness outputs results in [EARL](https://www.w3.org/TR/EARL10-Schema/) (Evaluation and Report Language), an RDF vocabulary:

```turtle
@prefix earl: <http://www.w3.org/ns/earl#>.
@prefix dc: <http://purl.org/dc/terms/>.

<#test-result-001>
    a earl:TestResult ;
    earl:outcome earl:passed ;
    earl:subject <https://localhost:3000> ;
    earl:test <https://solid.github.io/conformance-test-harness/tests/protocol/read-resource> ;
    dc:date "2026-03-02T10:00:00Z"^^xsd:dateTime .
```

### Result States

| State | Meaning |
|-------|---------|
| `earl:passed` | Server correctly implements the behaviour |
| `earl:failed` | Server does not implement the behaviour |
| `earl:cantTell` | Test could not determine conformance |
| `earl:inapplicable` | Test does not apply to this configuration |
| `earl:untested` | Test was not run |

---

## Test Categories

### Protocol Tests

Verifies Solid Protocol (https://solidproject.org/TR/protocol) compliance:

- Resource reading (GET) — Turtle, JSON-LD content negotiation
- Resource writing (PUT, POST, PATCH with N3 Patch)
- Resource deletion (DELETE)
- Container containment triples auto-generated
- Auxiliary resources (ACL, description)
- HTTP headers (`Link`, `ETag`, `WAC-Allow`)
- Error responses (400, 401, 403, 404, 405, 409, 412)

### WAC Tests

Verifies Web Access Control implementation:

- Default ACL inheritance
- Agent-specific access grants
- `acl:agentGroup` support
- `acl:origin` restrictions
- Public access (`acl:agentClass foaf:Agent`)
- Authenticated-only access (`acl:agentClass acl:AuthenticatedAgent`)

### ACP Tests

Verifies Access Control Policy implementation (see [solid/spec.md](../solid/spec.md)):

- Access Control Resources (ACRs)
- Policy and matcher evaluation
- `acp:allOf`, `acp:anyOf`, `acp:noneOf` logic
- Special agents: `acp:PublicAgent`, `acp:AuthenticatedAgent`
- Deny-overrides-allow logic

### Notifications Tests

Verifies the Solid Notifications Protocol:

- WebSocket channel subscription
- Resource update notifications
- Create/update/delete event types

---

## Writing Custom Test Cases

The harness supports extending with custom tests in TypeScript using `@inrupt/solid-client` and `solid-client-authn-node`.

```typescript
import { Session } from "@inrupt/solid-client-authn-node";
import { getSolidDataset, getThing, getUrl } from "@inrupt/solid-client";
import { expect, test } from "@playwright/test";

test("server advertises storage in WebID profile", async () => {
  const session = new Session();
  await session.login({
    oidcIssuer: process.env.ALICE_IDP!,
    clientId: process.env.ALICE_CLIENT_ID!,
    clientSecret: process.env.ALICE_CLIENT_SECRET!,
  });

  const profile = await getSolidDataset(process.env.ALICE_WEBID!, {
    fetch: session.fetch,
  });

  const profileThing = getThing(profile, process.env.ALICE_WEBID!);
  const storage = getUrl(profileThing, "http://www.w3.org/ns/pim/space#storage");

  expect(storage).not.toBeNull();
  await session.logout();
});
```

---

## Common Conformance Failures

| Failure | Spec Reference | Common Cause |
|---------|---------------|-------------|
| Missing `Link: <acl>; rel="acl"` header | Solid Protocol §3 | ACL support not enabled |
| PATCH returns 415 for `text/n3` | Solid Protocol §4 | N3 Patch not supported |
| POST doesn't return `Location` header | Solid Protocol §3 | Custom implementation missing header |
| Container triples not auto-generated | Solid Protocol §3.3 | Non-standard container implementation |
| `solid:oidcIssuer` not in profile | WebID Profile | Profile not populated on server registration |
| DPoP validation fails | Solid-OIDC | Outdated or incorrect DPoP implementation |
| ACP deny policy not overriding allow | ACP §3 | Allow logic evaluated before deny |

---

## CI Integration

```yaml
# .github/workflows/conformance.yml
name: Solid Conformance Tests

on:
  push:
    branches: [main]
  schedule:
    - cron: "0 2 * * *"  # nightly

jobs:
  conformance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start CSS
        run: |
          npx @solid/community-server \
            -c @css:config/file.json \
            -f ./test-data \
            --port 3000 &
          sleep 5

      - name: Provision test accounts
        run: node scripts/provision-test-accounts.js

      - name: Clone and run conformance harness
        run: |
          git clone https://github.com/solid-contrib/conformance-test-harness harness
          cd harness && npm install
          SERVER_ROOT=http://localhost:3000 \
          ALICE_WEBID=http://localhost:3000/alice/profile/card#me \
          ALICE_CLIENT_ID=${{ secrets.ALICE_CLIENT_ID }} \
          ALICE_CLIENT_SECRET=${{ secrets.ALICE_CLIENT_SECRET }} \
          npm test -- --reporter html --output-dir ../conformance-report

      - uses: actions/upload-artifact@v4
        with:
          name: conformance-report
          path: conformance-report/

      - name: Fail on conformance regressions
        run: node scripts/check-conformance.js conformance-report/results.json
```

---

## Conformance Test Checklist

- [ ] Server URL and test user credentials configured
- [ ] Test user WebIDs resolvable and contain `solid:oidcIssuer`
- [ ] Test user pods provisioned with appropriate initial content
- [ ] Full harness suite runs without infrastructure errors
- [ ] All `earl:failed` results reviewed against spec
- [ ] Known failures documented with issue tracker references
- [ ] Regression: compare results against previous run
- [ ] EARL report archived for audit trail
- [ ] Conformance report shared with server implementation team

---

## Key Links

| Resource | URL |
|----------|-----|
| Conformance Test Harness | https://github.com/solid-contrib/conformance-test-harness |
| Inrupt CTH Blog | https://www.inrupt.com/blog/conformance-test-suite |
| Solid Protocol | https://solidproject.org/TR/protocol |
| EARL Vocabulary | https://www.w3.org/TR/EARL10-Schema/ |
| CSS (test server) | https://github.com/CommunitySolidServer/CommunitySolidServer |
| Solid Spec Overview | See [solid/spec.md](../solid/spec.md) |
