# Solid Specifications Skill

You are an expert on the Solid Protocol and its companion specifications. Use this knowledge to help developers understand, implement, and debug Solid-compliant applications.

## Core Specifications

| Spec | Version | URL | Purpose |
|------|---------|-----|---------|
| Solid Protocol | v0.11.0 | https://solidproject.org/TR/protocol | Core data pod and resource access rules |
| WebID Profile | v1.0.0 | https://solid.github.io/webid-profile/ | Identity and agent description |
| Solid-OIDC | v0.1.0 | https://solidproject.org/TR/oidc | Decentralised authentication |
| Access Control Policy (ACP) | v0.9.0 | https://solidproject.org/TR/acp | Fine-grained authorisation |

---

## 1. Solid Protocol

**What it is**: The foundation spec defining how data pods expose resources over HTTP using Linked Data principles.

### Key Concepts

**Storage**: A space of URIs giving agents controlled access to resources. Each storage has a root container.

**Resources and Containers**: Resources live at HTTP URIs. Container URIs end with `/`. Containers hold other resources and containers, forming a hierarchy.

**WebID**: An HTTP URI that identifies an agent. Dereferencing it returns an RDF document describing them.

### HTTP Behaviour

- Servers MUST support GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
- All write requests MUST include `Content-Type`; missing headers → 400
- Servers MUST support Turtle (`text/turtle`) and JSON-LD (`application/ld+json`)
- HTTPS is required; HTTP → HTTPS redirect if both schemes are exposed

### Content Negotiation

```
GET /profile/card HTTP/1.1
Accept: text/turtle
```

Servers must honour `Accept` headers and respond with matching RDF serialisation.

### PATCH with N3

```
PATCH /data/notes.ttl HTTP/1.1
Content-Type: text/n3

@prefix solid: <http://www.w3.org/ns/solid/terms#>.
<> a solid:InsertDeletePatch;
  solid:inserts { <#note1> <http://schema.org/text> "Hello world" . }.
```

- 409 Conflict if conditions produce zero or multiple matches.

### Auxiliary Resources

- **ACL resources**: control access (`Link: <acl>; rel="acl"`)
- **Description resources**: metadata about a resource (`Link: <meta>; rel="describedby"`)
- Auxiliary resources are deleted automatically when their parent is deleted.

### Notifications

The Solid Notifications Protocol allows clients to subscribe to resource changes. Servers act as Resource Server, Subscription Server, Notification Sender, and Notification Receiver.

### Common Mistakes

- Forgetting trailing `/` on container URIs
- Not sending `Content-Type` on PATCH/PUT
- Using WAC and ACP interchangeably — servers implement one or both, check which
- Assuming CORS is sufficient for access control — always use ACL/ACP

---

## 2. WebID Profile

**What it is**: An RDF document at a WebID URI describing a social agent's identity, storage locations, and preferences.

### Minimum Required Triples

```turtle
@prefix foaf: <http://xmlns.com/foaf/0.1/>.
@prefix pim:  <http://www.w3.org/ns/pim/space#>.

<https://example.solidcommunity.net/profile/card#me>
    a foaf:Agent ;
    pim:preferencesFile <https://example.solidcommunity.net/settings/prefs.ttl> ;
    pim:storage <https://example.solidcommunity.net/> .
```

### Key Predicates

| Predicate | Required | Description |
|-----------|----------|-------------|
| `rdf:type foaf:Agent` | Yes | Identifies the subject as an agent |
| `pim:preferencesFile` | Yes | Pointer to private preferences document |
| `pim:storage` | Recommended | Location of the agent's pod storage |
| `solid:oidcIssuer` | Recommended | The OIDC Identity Provider for this WebID |
| `ldp:inbox` | Optional | Inbox for incoming notifications |
| `rdfs:seeAlso` | Optional | Links to extended profile documents |

### Preferences Document

- Must be typed as `pim:ConfigurationFile`
- Private (not publicly readable)
- Holds app settings and personalisation data

### Type Index

- Public and private type index documents link resource types to storage locations
- Discovered via the WebID profile
- Enables apps to find where a user stores specific types of data

### Modifying a Profile

Use PATCH (N3 Patch) for targeted updates. Servers protect `solid:oidcIssuer` — attempting to change it returns 409.

```turtle
# Read profile
GET /profile/card HTTP/1.1
Accept: text/turtle

# Patch profile to add name
PATCH /profile/card HTTP/1.1
Content-Type: text/n3

@prefix solid: <http://www.w3.org/ns/solid/terms#>.
@prefix foaf: <http://xmlns.com/foaf/0.1/>.
<> a solid:InsertDeletePatch;
  solid:inserts { <#me> foaf:name "Alice" . }.
```

---

## 3. Solid-OIDC

**What it is**: An authentication layer built on OAuth 2.0 + OpenID Connect that works without pre-established trust between resource servers and identity providers.

### Flow Overview

1. App redirects user to their OIDC Identity Provider (IdP)
2. IdP issues a DPoP-bound ID Token containing the `webid` claim
3. App presents the ID Token + DPoP proof to the Resource Server
4. Resource Server validates the token by checking the WebID profile for `solid:oidcIssuer`

### Authorization Code Flow with PKCE

This is the **mandatory** flow. Never use Implicit Flow.

```
1. App generates code_verifier + code_challenge (PKCE)
2. Redirect → IdP /authorize?response_type=code&code_challenge=...
3. IdP issues auth code
4. App exchanges code for tokens at /token endpoint
5. App receives id_token + access_token (DPoP-bound)
```

### DPoP-Bound ID Token Claims

```json
{
  "webid": "https://alice.solidcommunity.net/profile/card#me",
  "iss": "https://solidcommunity.net",
  "aud": ["https://myapp.example", "solid"],
  "cnf": { "jkt": "<DPoP public key thumbprint>" },
  "exp": 1700000000,
  "iat": 1699996400
}
```

### DPoP Proof Header

Every resource request must include a DPoP proof JWT:

```
Authorization: DPoP <token>
DPoP: <proof-jwt>
```

### Client Identification

Two options:
1. **Dereferenceable URI** — client ID is a URL that returns a JSON-LD client registration document
2. **Traditional OIDC registration** — dynamic or static client registration (optional in decentralised context)

### WebID Issuer Validation

Resource servers must verify that the `iss` claim in the ID Token matches a `solid:oidcIssuer` triple in the user's WebID Profile. This prevents token misuse.

### Common Mistakes

- Using Implicit Flow (not supported)
- Forgetting to include `"solid"` in the `aud` claim
- Not binding tokens with DPoP
- Checking the wrong WebID for issuer validation

---

## 4. Access Control Policy (ACP)

**What it is**: An RDF-based language for defining who can access what on a Solid pod, using a deny-overrides policy model.

### Core Objects

```
Resource ──has ACR──▶ Access Control Resource (ACR)
                              │
                        Access Control
                              │
                           Policy
                         ┌────┴────┐
                      Allow     Deny
                         │         │
                       Matcher  Matcher
```

### Access Control Resource (ACR)

Each resource has an associated ACR linked via:

```
Link: <resource.acr>; rel="http://www.w3.org/ns/solid/acp#accessControl"
```

### Matchers

Matchers define conditions for a policy. They evaluate against the **request context** (agent, client, issuer, target resource).

```turtle
@prefix acp: <http://www.w3.org/ns/solid/acp#>.

<#matcher-alice>
    a acp:Matcher ;
    acp:agent <https://alice.solidcommunity.net/profile/card#me> .
```

Special agents:
- `acp:PublicAgent` — anyone (including unauthenticated)
- `acp:AuthenticatedAgent` — any authenticated user
- `acp:CreatorAgent` — whoever created the resource
- `acp:OwnerAgent` — the pod owner

### Policies

```turtle
<#policy-read>
    a acp:Policy ;
    acp:allow acp:Read ;
    acp:anyOf <#matcher-alice> .
```

Policy satisfaction logic:

| Condition | Meaning |
|-----------|---------|
| `acp:allOf` | ALL listed matchers must match |
| `acp:anyOf` | AT LEAST ONE listed matcher must match |
| `acp:noneOf` | NONE of the listed matchers must match |

### Access Control (connecting ACR to Policy)

```turtle
<#access-control>
    a acp:AccessControl ;
    acp:apply <#policy-read> .
```

### Grant Logic

Access mode is granted **if and only if**:
- At least one satisfied policy **allows** it, AND
- No satisfied policy **denies** it

**Deny overrides allow.**

### Common Access Patterns

**Public read access:**
```turtle
<#matcher-public> a acp:Matcher ; acp:agent acp:PublicAgent .
<#policy-public>  a acp:Policy  ; acp:allow acp:Read ; acp:anyOf <#matcher-public> .
```

**Authenticated write access:**
```turtle
<#matcher-auth> a acp:Matcher ; acp:agent acp:AuthenticatedAgent .
<#policy-write> a acp:Policy  ; acp:allow acp:Write, acp:Append ; acp:anyOf <#matcher-auth> .
```

### ACP vs WAC

| | ACP | WAC |
|-|-----|-----|
| Model | Policy + Matchers (RDF) | ACL rules (RDF) |
| Deny support | Yes (deny overrides) | Limited |
| Granularity | Fine-grained, composable | Basic |
| Spec status | Active development | Older, widely deployed |

---

## Quick Reference: Spec URLs

- Solid Protocol: https://solidproject.org/TR/protocol
- WebID Profile: https://solid.github.io/webid-profile/
- Solid-OIDC: https://solidproject.org/TR/oidc
- ACP: https://solidproject.org/TR/acp
- Solid Notifications: https://solidproject.org/TR/notifications-protocol
- Web Access Control: https://solidproject.org/TR/wac

---

## Other awesome skills

For more Claude/LLM skills (document processing, dev tools, automation, MCP, etc.), see [Awesome Claude Skills](https://github.com/ComposioHQ/awesome-claude-skills).
