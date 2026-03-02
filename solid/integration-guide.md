# Solid Integration Guide Skill

You are an expert on integrating JavaScript/TypeScript applications with the Solid ecosystem. You know the Inrupt SDK, LDO, and solid-client-authn-js libraries in depth, and can guide developers through authentication, reading, writing, and managing access to Solid pods.

Based on application being built by user, recommend library to use and get confirmation from the user.

---

## Library Overview

| Library | Package | Best for |
|---------|---------|---------|
| @inrupt/solid-client | `@inrupt/solid-client` | Reading/writing structured RDF data (SolidDataset/Thing API) |
| solid-client-authn-browser | `@inrupt/solid-client-authn-browser` | Browser authentication (OIDC + DPoP) |
| solid-client-authn-node | `@inrupt/solid-client-authn-node` | Node.js / server-side authentication |
| LDO | `ldo` / `@ldo/solid` | TypeScript-first RDF with ShEx shapes; type-safe linked data |
| @ldo/connected-solid | `@ldo/connected-solid` | LDO + Solid pods: containers, WAC, live updates |
| N3.js | `n3` | Low-level RDF parsing, serialising, and in-memory quad storage |

---

## Authentication with solid-client-authn-js

**Docs**: https://inrupt.github.io/solid-client-authn-js/
**Repo**: https://github.com/inrupt/solid-client-authn-js

### Browser Authentication

```bash
npm install @inrupt/solid-client-authn-browser
```

```typescript
import {
  login,
  logout,
  handleIncomingRedirect,
  getDefaultSession,
  fetch,                      // authenticated fetch — use this for all pod requests
} from "@inrupt/solid-client-authn-browser";

// 1. Handle redirect on page load (restore session after OIDC redirect)
await handleIncomingRedirect({ restorePreviousSession: true });

// 2. Check if already logged in
const session = getDefaultSession();
if (!session.info.isLoggedIn) {
  // 3. Redirect user to their OIDC provider
  await login({
    oidcIssuer: "https://solidcommunity.net",
    redirectUrl: window.location.href,
    clientName: "My Solid App",
  });
}

// 4. Use the authenticated session's fetch for all pod requests
const response = await fetch("https://alice.solidcommunity.net/profile/card");
```

### Node.js Authentication (Client Credentials)

```bash
npm install @inrupt/solid-client-authn-node
```

```typescript
import { Session } from "@inrupt/solid-client-authn-node";

const session = new Session();

await session.login({
  oidcIssuer: "https://solidcommunity.net",
  clientId: "your-client-id",
  clientSecret: "your-client-secret",
  clientName: "My Node App",
});

// Use session.fetch for authenticated requests
const response = await session.fetch("https://alice.solidcommunity.net/data/");
```

### Key Rules

- Always use the session's `fetch` (not global `fetch`) for requests to Solid pods
- Call `handleIncomingRedirect` on every page load to restore session after OIDC callback
- Store the `oidcIssuer` — prompt users to enter their pod provider if unknown
- Tokens are DPoP-bound; do not strip or modify Authorization headers

---

## Reading and Writing with @inrupt/solid-client

**Docs**: https://docs.inrupt.com/developer-tools/javascript/client-libraries/
**Package**: `@inrupt/solid-client`

```bash
npm install @inrupt/solid-client @inrupt/solid-client-authn-browser
```

### Core Concepts

| Concept | Description |
|---------|-------------|
| `SolidDataset` | An RDF document fetched from a pod |
| `Thing` | A named set of RDF triples within a SolidDataset (identified by URL) |
| `saveSolidDatasetAt` | Write a modified SolidDataset back to the pod |
| `getThing` / `setThing` | Read/modify a Thing within a dataset |
| `buildThing` / `createThing` | Construct new Things |

### Read a Resource

```typescript
import {
  getSolidDataset,
  getThing,
  getStringNoLocale,
  getUrl,
} from "@inrupt/solid-client";
import { fetch } from "@inrupt/solid-client-authn-browser";
import { FOAF, SCHEMA_INRUPT } from "@inrupt/vocab-common-rdf";

const dataset = await getSolidDataset(
  "https://alice.solidcommunity.net/profile/card",
  { fetch }
);

const profile = getThing(
  dataset,
  "https://alice.solidcommunity.net/profile/card#me"
);

const name = getStringNoLocale(profile, FOAF.name);
const storage = getUrl(profile, "http://www.w3.org/ns/pim/space#storage");
```

### Write a Resource

```typescript
import {
  getSolidDataset,
  getThing,
  setThing,
  setStringNoLocale,
  saveSolidDatasetAt,
  buildThing,
  createThing,
} from "@inrupt/solid-client";
import { fetch } from "@inrupt/solid-client-authn-browser";
import { SCHEMA_INRUPT } from "@inrupt/vocab-common-rdf";

// Fetch existing dataset
const dataset = await getSolidDataset(
  "https://alice.solidcommunity.net/notes/note1.ttl",
  { fetch }
);

// Modify an existing Thing
let note = getThing(
  dataset,
  "https://alice.solidcommunity.net/notes/note1.ttl#note"
);
note = setStringNoLocale(note, SCHEMA_INRUPT.text, "Updated note content");

// Save back to the pod
const updatedDataset = setThing(dataset, note);
await saveSolidDatasetAt(
  "https://alice.solidcommunity.net/notes/note1.ttl",
  updatedDataset,
  { fetch }
);
```

### Create a New Resource

```typescript
import {
  createSolidDataset,
  setThing,
  buildThing,
  createThing,
  saveSolidDatasetAt,
} from "@inrupt/solid-client";
import { fetch } from "@inrupt/solid-client-authn-browser";
import { RDF, SCHEMA_INRUPT } from "@inrupt/vocab-common-rdf";

const newNote = buildThing(createThing({ name: "my-note" }))
  .addUrl(RDF.type, "https://schema.org/TextObject")
  .addStringNoLocale(SCHEMA_INRUPT.text, "My first note")
  .build();

const dataset = setThing(createSolidDataset(), newNote);

// saveSolidDatasetAt creates intermediate containers automatically
await saveSolidDatasetAt(
  "https://alice.solidcommunity.net/notes/my-note.ttl",
  dataset,
  { fetch }
);
```

### ACL Management

```typescript
import {
  getSolidDatasetWithAcl,
  getResourceAcl,
  setAgentResourceAccess,
  saveAclFor,
} from "@inrupt/solid-client";

const datasetWithAcl = await getSolidDatasetWithAcl(resourceUrl, { fetch });
const resourceAcl = getResourceAcl(datasetWithAcl);

const updatedAcl = setAgentResourceAccess(
  resourceAcl,
  "https://bob.solidcommunity.net/profile/card#me",
  { read: true, write: false, append: false, control: false }
);

await saveAclFor(datasetWithAcl, updatedAcl, { fetch });
```

---

## LDO (Linked Data Objects)

**Docs**: https://ldo.js.org/
**Package**: `ldo`, `@ldo/solid`, `@ldo/connected-solid`

LDO provides TypeScript-typed access to RDF data using ShEx shapes. It generates TypeScript interfaces from shapes, so you work with familiar object literals instead of raw RDF APIs.

### Install

```bash
npm install ldo @ldo/solid
npm install --save-dev @ldo/cli
```

### Generate TypeScript from ShEx

```bash
# Given a ShEx file at ./shapes/Person.shex
npx ldo generate ./shapes
```

This generates:
- TypeScript interfaces (`.typings.ts`)
- JSON-LD context (`.context.ts`)
- Shape type (`.shapeTypes.ts`)

### Read RDF as TypeScript Object

```typescript
import { parseRdf } from "ldo";
import { PersonShapeType } from "./shapes/PersonShapeType";

const turtle = `
@prefix foaf: <http://xmlns.com/foaf/0.1/>.
<https://alice.example.com/#me> a foaf:Person ; foaf:name "Alice" .
`;

const dataset = await parseRdf(turtle, { baseIRI: "https://alice.example.com/" });
const ldoDataset = createLdoDataset([...dataset]);

const person = ldoDataset
  .usingType(PersonShapeType)
  .fromSubject("https://alice.example.com/#me");

console.log(person.name); // "Alice"
```

### Write via LDO

```typescript
const [person, setThing] = createLdoDataset().usingType(PersonShapeType).write();
person.["@id"] = "https://alice.example.com/#me";
person.name = "Alice Updated";

// Serialize back to Turtle
const turtle = await serialize(dataset, { format: "text/turtle" });
```

### @ldo/connected-solid — LDO + Solid Pods

Combines LDO with Solid pods for type-safe, live pod interaction:

```typescript
import { createSolidLdoDataset } from "@ldo/connected-solid";

const solidLdoDataset = createSolidLdoDataset();
await solidLdoDataset.auth.login({ oidcIssuer: "https://solidcommunity.net" });

// Read a resource into the dataset
await solidLdoDataset.getResource("https://alice.solidcommunity.net/notes/note1.ttl").read();

// Access typed object
const note = solidLdoDataset
  .usingType(NoteShapeType)
  .fromSubject("https://alice.solidcommunity.net/notes/note1.ttl#note");

// Mutate and commit
note.text = "Updated via LDO";
await solidLdoDataset.commitChanges();
```

**Additional @ldo/connected-solid features:**
- Container listing and creation
- WAC access control management
- Live update subscriptions (WebSocket notifications)

---

## N3.js — Low-Level RDF Parsing and Storage

**Repo**: https://github.com/rdfjs/N3.js
**Docs**: https://rdf.js.org/N3.js/
**Package**: `n3`

N3.js is a low-level, spec-compliant RDF library for JavaScript. It handles parsing, serialising, and in-memory storage of RDF quads. It implements the [RDF/JS interfaces](https://rdf.js.org/) and is the foundation other libraries build on.

Use N3.js when you need direct control over RDF data — raw PATCH body construction, custom parsers, graph reasoning, or working with formats the higher-level libraries don't expose.

### Install

```bash
npm install n3
```

```typescript
import * as N3 from "n3";
// or specific imports:
import { Parser, Writer, Store, DataFactory, StreamParser, StreamWriter } from "n3";
const { namedNode, literal, defaultGraph, quad } = DataFactory;
```

### DataFactory — Constructing RDF Terms

```typescript
import { DataFactory } from "n3";
const { namedNode, literal, blankNode, defaultGraph, quad } = DataFactory;

// Named node (IRI)
const alice = namedNode("https://alice.solidcommunity.net/profile/card#me");

// Literal with language tag
const name = literal("Alice", "en");

// Literal with datatype
import { xsd } from "@tpluscode/rdf-ns-builders";
const dob = literal("1990-01-01", namedNode("http://www.w3.org/2001/XMLSchema#date"));

// Build a quad (subject, predicate, object, graph)
const myQuad = quad(
  alice,
  namedNode("http://xmlns.com/foaf/0.1/name"),
  name,
  defaultGraph()
);

console.log(myQuad.subject.value); // "https://alice.solidcommunity.net/profile/card#me"
```

### Parser — Turtle / N-Triples / TriG / N-Quads to Quads

```typescript
import { Parser } from "n3";

const turtle = `
  @prefix foaf: <http://xmlns.com/foaf/0.1/>.
  <https://alice.example.com/#me> foaf:name "Alice" .
`;

// Callback-based (async, streaming-friendly)
const parser = new Parser({ baseIRI: "https://alice.example.com/" });
parser.parse(turtle, (error, quad, prefixes) => {
  if (quad) {
    console.log(quad.subject.value, quad.predicate.value, quad.object.value);
  } else {
    console.log("Done. Prefixes:", prefixes);
  }
});

// Synchronous (returns array)
const quads = new Parser().parse(turtle);
```

**Format options** (pass as `{ format: "..." }`):

| String | Parses |
|--------|--------|
| `"Turtle"` (default) | Turtle + permissive superset |
| `"N-Triples"` | N-Triples |
| `"TriG"` | TriG (named graphs) |
| `"N-Quads"` | N-Quads |
| `"N3"` / `"text/n3"` | N3 (with rules) |

### Writer — Quads to Turtle / N-Triples / TriG

```typescript
import { Writer, DataFactory } from "n3";
const { namedNode, literal } = DataFactory;

// Write to string
const writer = new Writer({
  prefixes: { foaf: "http://xmlns.com/foaf/0.1/" },
  // format: "N-Triples"  // override default Turtle
});

writer.addQuad(
  namedNode("https://alice.example.com/#me"),
  namedNode("http://xmlns.com/foaf/0.1/name"),
  literal("Alice", "en")
);

writer.end((error, result) => {
  console.log(result);
  // @prefix foaf: <http://xmlns.com/foaf/0.1/>.
  // <https://alice.example.com/#me> foaf:name "Alice"@en.
});
```

### Store — In-Memory Quad Index

`N3.Store` is an efficient indexed in-memory store with quad-pattern lookups.

```typescript
import { Store, Parser, DataFactory } from "n3";
const { namedNode } = DataFactory;

const store = new Store();

// Add quads directly
store.addQuad(
  namedNode("https://alice.example.com/#me"),
  namedNode("http://xmlns.com/foaf/0.1/name"),
  DataFactory.literal("Alice")
);

// Or parse directly into the store
const parser = new Parser();
const quads = parser.parse(`
  @prefix foaf: <http://xmlns.com/foaf/0.1/>.
  <https://alice.example.com/#me> foaf:name "Alice" ; foaf:knows <https://bob.example.com/#me> .
`);
store.addQuads(quads);

// Query: getQuads(subject, predicate, object, graph) — null = wildcard
const names = store.getQuads(
  namedNode("https://alice.example.com/#me"),
  namedNode("http://xmlns.com/foaf/0.1/name"),
  null,
  null
);
console.log(names[0].object.value); // "Alice"

// Count matching quads
const count = store.countQuads(null, namedNode("http://xmlns.com/foaf/0.1/knows"), null, null);

// Iterate over unique subjects
store.forSubjects(
  (subject) => console.log(subject.value),
  namedNode("http://xmlns.com/foaf/0.1/knows"),
  null,
  null
);

// Remove quads
store.removeQuads(names);
// or by pattern:
store.removeMatches(namedNode("https://alice.example.com/#me"), null, null, null);
```

### StreamParser and StreamWriter — Large Files

```typescript
import { StreamParser, StreamWriter } from "n3";
import { createReadStream, createWriteStream } from "fs";

// Parse a large Turtle file as a stream
const inputStream = createReadStream("data.ttl");
const streamParser = new StreamParser({ format: "Turtle" });

inputStream.pipe(streamParser);

streamParser.on("data", (quad) => {
  console.log(quad.subject.value);
});
streamParser.on("end", () => console.log("Done"));

// Transform stream: re-serialise to N-Triples
const streamWriter = new StreamWriter({ format: "N-Triples" });
inputStream
  .pipe(new StreamParser())
  .pipe(streamWriter)
  .pipe(createWriteStream("output.nt"));
```

### SPARQL Queries with Comunica

N3.js does not include a SPARQL engine, but its `Store` implements the RDF/JS interfaces so it works directly with [Comunica](https://comunica.dev/).

```bash
npm install @comunica/query-sparql-rdfjs
```

```typescript
import { QueryEngine } from "@comunica/query-sparql-rdfjs";
import { Store, Parser } from "n3";

const store = new Store();
store.addQuads(new Parser().parse(`
  @prefix foaf: <http://xmlns.com/foaf/0.1/>.
  <#alice> foaf:name "Alice" ; foaf:knows <#bob> .
  <#bob>   foaf:name "Bob" .
`));

const engine = new QueryEngine();
const bindingsStream = await engine.queryBindings(
  "SELECT ?name WHERE { ?s <http://xmlns.com/foaf/0.1/name> ?name }",
  { sources: [store] }
);

bindingsStream.on("data", (bindings) => {
  console.log(bindings.get("name").value);
});
```

### N3.js in a Solid Context

N3.js is useful when working directly with Solid at the HTTP level — parsing response bodies, constructing N3 PATCH payloads, or processing raw Turtle resources.

```typescript
import { fetch } from "@inrupt/solid-client-authn-browser";
import { Parser, Writer, Store, DataFactory } from "n3";
const { namedNode, literal } = DataFactory;

// Fetch and parse a Solid resource as raw RDF
const response = await fetch("https://alice.solidcommunity.net/profile/card", {
  headers: { Accept: "text/turtle" },
});
const turtle = await response.text();

const store = new Store();
store.addQuads(new Parser({ baseIRI: response.url }).parse(turtle));

// Query the profile
const names = store.getQuads(
  null,
  namedNode("http://xmlns.com/foaf/0.1/name"),
  null,
  null
);
console.log(names.map((q) => q.object.value));

// Build an N3 Patch body
const patchWriter = new Writer({ format: "text/n3" });
patchWriter.addQuad(
  namedNode(""),
  namedNode("http://www.w3.org/1999/02/22-rdf-syntax-ns#type"),
  namedNode("http://www.w3.org/ns/solid/terms#InsertDeletePatch")
);
// ... add solid:inserts quads
patchWriter.end(async (err, patchBody) => {
  await fetch("https://alice.solidcommunity.net/profile/card", {
    method: "PATCH",
    headers: { "Content-Type": "text/n3" },
    body: patchBody,
  });
});
```

---

## Choosing the Right Library

| Scenario | Recommended Library |
|----------|-------------------|
| Browser app, structured data | `@inrupt/solid-client` + `solid-client-authn-browser` |
| Node.js script or API | `@inrupt/solid-client` + `solid-client-authn-node` |
| TypeScript-first, shape-driven | `ldo` + `@ldo/solid` |
| React app with live updates | `@ldo/connected-solid` |
| Raw HTTP, no library overhead | Native `fetch` with `solid-client-authn` session fetch |
| Low-level RDF parsing / serialising | `n3` (N3.js) |
| In-memory graph querying | `n3` Store + Comunica |
| Custom PATCH body construction | `n3` Writer (`text/n3` format) |
| Stream processing large RDF files | `n3` StreamParser + StreamWriter |

---

## Common Patterns

### Check if User is Logged In

```typescript
import { getDefaultSession } from "@inrupt/solid-client-authn-browser";

const { isLoggedIn, webId } = getDefaultSession().info;
```

### Discover User's Storage

```typescript
import { getSolidDataset, getThing, getUrl } from "@inrupt/solid-client";

const profile = await getSolidDataset(webId, { fetch });
const profileThing = getThing(profile, webId);
const storageUrl = getUrl(profileThing, "http://www.w3.org/ns/pim/space#storage");
```

For robust pod root discovery (handles servers that don't advertise `pim:storage` on the profile), use the recursive `getPodRoot` pattern from [Bashlib util.ts L73-104](https://github.com/SolidLabResearch/Bashlib/blob/80de25cbb4b3ed057f95e25bc057f1be9b00cef3/src/utils/util.ts#L73-L104). It uses a three-stage approach:

1. HEAD request — checks the `Link` header for a `pim:Storage` relation on the resource
2. Dataset query — fetches the resource and looks for a `pim:storage` triple
3. Recursive traversal — walks up the URL path one segment at a time, repeating steps 1–2

```typescript
async function getPodRoot(url: string, fetch: typeof globalThis.fetch): Promise<string | null> {
  // Stage 1: check Link header
  const headRes = await fetch(url, { method: "HEAD" });
  const linkHeader = headRes.headers.get("Link") ?? "";
  if (linkHeader.includes("http://www.w3.org/ns/pim/space#Storage")) {
    return url;
  }

  // Stage 2: check dataset for pim:storage triple
  try {
    const dataset = await getSolidDataset(url, { fetch });
    const thing = getThing(dataset, url);
    const storage = getUrl(thing, "http://www.w3.org/ns/pim/space#storage");
    if (storage) return storage;
  } catch {}

  // Stage 3: recurse up the URL path
  const parent = new URL(url);
  const segments = parent.pathname.replace(/\/$/, "").split("/");
  if (segments.length <= 1) return null;
  parent.pathname = segments.slice(0, -1).join("/") + "/";
  return getPodRoot(parent.toString(), fetch);
}
```

### List a Container

```typescript
import { getSolidDataset, getContainedResourceUrlAll } from "@inrupt/solid-client";

const container = await getSolidDataset(containerUrl, { fetch });
const resourceUrls = getContainedResourceUrlAll(container);
```

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| 401 Unauthorized | Not using session `fetch` | Import `fetch` from authn library; pass to all solid-client calls |
| 403 Forbidden | Insufficient ACL/ACP | Check access control on the resource |
| 409 Conflict on save | Outdated dataset / ETag mismatch | Re-fetch the dataset before saving |
| CORS error in browser | Server CORS config | Use a server that supports CORS or proxy in dev |
| Session lost on refresh | `handleIncomingRedirect` not called | Call with `restorePreviousSession: true` on every page load |
| DPoP error | Token not DPoP-bound | Always use the authn library's `fetch`; never strip headers |

---

## Key Links

| Resource | URL |
|----------|-----|
| Inrupt SDK docs | https://docs.inrupt.com/sdk/javascript-sdk |
| @inrupt/solid-client npm | https://www.npmjs.com/package/@inrupt/solid-client |
| solid-client-authn-js | https://inrupt.github.io/solid-client-authn-js/ |
| solid-client-authn-js repo | https://github.com/inrupt/solid-client-authn-js |
| LDO docs | https://ldo.js.org/1.0.0-alpha.X/ |
| @ldo/connected-solid | https://ldo.js.org/1.0.0-alpha.X/api/connected-solid/ |
| N3.js repo | https://github.com/rdfjs/N3.js |
| N3.js docs | https://rdf.js.org/N3.js/ |
| Comunica (SPARQL over N3 Store) | https://comunica.dev/docs/query/advanced/rdfjs_querying/ |
