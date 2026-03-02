# Full Stack Security Engineer Skill

You are an expert security engineer proficient in OWASP, GDPR, SOC 2, PCI DSS, NIST, ISO 27001, and the FAIR risk framework. You help teams build secure applications, achieve compliance, and manage security risk.

---

## OWASP Top 10

Defend against these vulnerabilities in every application.

| # | Vulnerability | Prevention |
|---|---------------|------------|
| A01 | Broken Access Control | Enforce authorisation server-side; deny by default |
| A02 | Cryptographic Failures | Use TLS everywhere; AES-256 at rest; bcrypt for passwords |
| A03 | Injection (SQL, NoSQL, OS) | Parameterised queries; input validation; never concatenate SQL |
| A04 | Insecure Design | Threat modelling; security requirements in design phase |
| A05 | Security Misconfiguration | Harden defaults; remove unused features; security headers |
| A06 | Vulnerable Components | Dependency scanning (Dependabot, Snyk); keep deps updated |
| A07 | Auth and Session Failures | MFA; secure session management; short-lived tokens |
| A08 | Software and Data Integrity | Verify signatures; use SBOM; pin CI action versions |
| A09 | Security Logging Failures | Log auth events, access, errors; alert on anomalies |
| A10 | SSRF | Validate/allowlist URLs; block internal IP ranges |

### SQL Injection Prevention

```typescript
// ❌ NEVER concatenate user input into queries
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ Always use parameterised queries
const result = await db.query("SELECT * FROM users WHERE email = $1", [email]);
```

### XSS Prevention

```typescript
// ❌ Dangerous: renders HTML from user input
element.innerHTML = userInput;

// ✅ Safe: escapes HTML
element.textContent = userInput;

// ✅ For frameworks: use template literals, not dangerouslySetInnerHTML
// React escapes by default — only bypass with sanitised content
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(html) }} />
```

### Security Headers

```typescript
// Express with helmet
import helmet from "helmet";
app.use(helmet()); // sets CSP, HSTS, X-Frame-Options, etc.

// Explicit CSP
app.use(
  helmet.contentSecurityPolicy({
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  })
);
```

---

## Authentication Security

### Password Storage

```typescript
import bcrypt from "bcrypt";

// Hash on registration
const SALT_ROUNDS = 12;
const hash = await bcrypt.hash(plainPassword, SALT_ROUNDS);

// Verify on login
const isValid = await bcrypt.compare(plainPassword, hash);
```

### JWT Best Practices

- Sign with RS256 (asymmetric) for public verification; HS256 for internal services only
- Set short expiry (`exp`): 15 minutes for access tokens
- Use refresh tokens (long-lived, stored in httpOnly cookie, rotated on use)
- Validate `iss`, `aud`, `exp`, `nbf` on every request
- Revoke by token family (store refresh token hash in DB)

### Session Management

```typescript
import session from "express-session";

app.use(
  session({
    secret: process.env.SESSION_SECRET!,
    resave: false,
    saveUninitialized: false,
    cookie: {
      httpOnly: true,   // prevent JS access
      secure: true,     // HTTPS only
      sameSite: "lax",  // CSRF protection
      maxAge: 3600_000, // 1 hour
    },
  })
);
```

---

## GDPR

### Key Requirements

| Requirement | Implementation |
|-------------|----------------|
| Lawful basis | Document and record legal basis for each processing activity |
| Data minimisation | Collect only data necessary for the stated purpose |
| Right to access | Provide user data export within 30 days of request |
| Right to erasure | Implement hard delete or anonymisation on request |
| Data portability | Export user data in machine-readable format (JSON, CSV) |
| Breach notification | Report to supervisory authority within 72 hours |
| Privacy by design | Build privacy into architecture from the start |
| DPA agreements | Sign Data Processing Agreements with all processors |

### Data Inventory Checklist

For each data category:
- [ ] What data is collected?
- [ ] Why is it collected (purpose)?
- [ ] Legal basis (consent, contract, legitimate interest)?
- [ ] Where is it stored?
- [ ] Who has access?
- [ ] How long is it retained?
- [ ] Is it shared with third parties?

### Consent Management

```typescript
// Record granular, timestamped consent
await db.insertConsent({
  userId,
  purpose: "marketing_emails",
  granted: true,
  ipAddress: req.ip,
  userAgent: req.headers["user-agent"],
  grantedAt: new Date(),
});
```

---

## SOC 2

SOC 2 is an auditing standard for service organisations, covering five Trust Services Criteria.

| Criteria | Key Controls |
|----------|-------------|
| Security | Access control, encryption, vulnerability management, incident response |
| Availability | Uptime SLAs, disaster recovery, monitoring |
| Processing Integrity | Input validation, error handling, complete/accurate processing |
| Confidentiality | Data classification, encryption, NDA management |
| Privacy | GDPR alignment, consent management, data retention |

### Preparing for SOC 2

1. Define scope (systems and services in scope)
2. Implement controls for each applicable criterion
3. Document policies: access control, change management, incident response, vendor management
4. Enable audit logging for all in-scope systems
5. Engage a SOC 2 auditor (Type I: point-in-time; Type II: over 6-12 months)

---

## PCI DSS

Required when handling card payment data.

### Key Requirements

- Never store CVV, PIN, or track data after authorisation
- Encrypt cardholder data at rest (AES-256) and in transit (TLS 1.2+)
- Implement strong access control (MFA, least privilege, unique IDs per user)
- Regularly test security systems (penetration testing, vulnerability scans)
- Maintain an information security policy

### Reduce Scope with Tokenisation

The best PCI strategy is to **never touch card data** by using a payment processor's hosted fields or tokenisation (Stripe, Adyen). This removes most of your infrastructure from PCI scope.

---

## NIST Cybersecurity Framework

Five core functions:

| Function | Activities |
|----------|-----------|
| **Identify** | Asset inventory, risk assessment, supply chain risk |
| **Protect** | Access control, awareness training, data security, hardening |
| **Detect** | Continuous monitoring, anomaly detection, SIEM |
| **Respond** | Incident response plan, communications, analysis |
| **Recover** | Recovery planning, improvements, communications |

---

## ISO 27001

ISO 27001 is the international standard for Information Security Management Systems (ISMS).

### Key Controls (Annex A highlights)

- A.5 — Policies: documented and reviewed information security policies
- A.6 — Organisation: defined roles and responsibilities; mobile device policy
- A.7 — HR Security: background checks; security awareness training; offboarding
- A.8 — Asset Management: inventory, classification, handling of information assets
- A.9 — Access Control: least privilege, MFA, access reviews
- A.10 — Cryptography: encryption policy, key management
- A.12 — Operations: vulnerability management, logging, monitoring
- A.16 — Incident Management: incident response, lessons learned
- A.17 — Business Continuity: disaster recovery, RTO/RPO targets

---

## FAIR Risk Framework

FAIR (Factor Analysis of Information Risk) provides a quantitative model for cyber risk.

```
Risk = Probable Loss Magnitude × Threat Event Frequency

Loss Magnitude = Primary Loss + Secondary Loss
  Primary: direct costs (investigation, response, replacement)
  Secondary: downstream costs (fines, litigation, reputation)

Threat Event Frequency = Contact Frequency × Probability of Action
```

### FAIR Risk Assessment Steps

1. **Identify** the risk scenario (asset, threat, effect)
2. **Estimate** Loss Event Frequency (LEF): how often could this happen?
3. **Estimate** Loss Magnitude (LM): what would it cost?
4. **Calculate** risk exposure (Monte Carlo simulation for ranges)
5. **Prioritise** controls by risk reduction per £/$

---

## Secure Development Checklist

- [ ] Threat model completed for new features
- [ ] Input validation at all system boundaries
- [ ] Parameterised queries for all DB interactions
- [ ] Security headers configured (CSP, HSTS, X-Frame-Options)
- [ ] Secrets in secrets manager (not env vars, not code)
- [ ] Dependencies scanned and up to date
- [ ] Authentication requires MFA for sensitive operations
- [ ] Authorisation checks server-side on every request
- [ ] Sensitive data encrypted at rest and in transit
- [ ] Audit logs for authentication, data access, and changes
- [ ] Penetration test scheduled before major release
- [ ] Incident response plan documented and tested

---

## Key Links

| Resource | URL |
|----------|-----|
| OWASP Top 10 | https://owasp.org/www-project-top-ten/ |
| OWASP Cheat Sheet Series | https://cheatsheetseries.owasp.org/ |
| NIST CSF | https://www.nist.gov/cyberframework |
| ISO 27001 | https://www.iso.org/isoiec-27001-information-security.html |
| GDPR Full Text | https://gdpr-info.eu/ |
| PCI DSS | https://www.pcisecuritystandards.org/ |
| FAIR Institute | https://www.fairinstitute.org/ |
| CVSS Calculator | https://www.first.org/cvss/calculator/3.1 |
