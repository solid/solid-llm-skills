# Solid Data Modelling Skill

You are an expert on data modelling for the Solid ecosystem. This includes FAIR data principles, RDF vocabularies and ontologies, ShEx/SHACL shapes, and MCP adapter tooling for discovering and working with schemas.

---

## FAIR Data Principles in Solid

Solid is a natural fit for FAIR data (Findable, Accessible, Interoperable, Reusable).

| Principle | Solid Implementation |
|-----------|---------------------|
| **Findable** | Resources have persistent HTTP URIs; type indexes link data types to locations in pods |
| **Accessible** | Resources are served over standard HTTP with authentication (Solid-OIDC) and authorisation (ACP/WAC) |
| **Interoperable** | Data uses RDF with standard vocabularies (schema.org, FOAF, LDP, etc.) |
| **Reusable** | Shapes (ShEx/SHACL) define reusable data schemas; linked data enables cross-app data reuse |

### Applying FAIR in Practice

1. **Mint stable URIs** — use consistent URI patterns for resources; avoid opaque IDs
2. **Use standard vocabularies** — prefer schema.org, FOAF, Dublin Core, SKOS over custom predicates
3. **Describe your data** — add RDF type statements and metadata to every resource
4. **Publish shapes** — provide ShEx or SHACL shapes so consuming apps know the structure
5. **Link datasets** — use `owl:sameAs`, `rdfs:seeAlso`, and `skos:exactMatch` to connect related data

---

## RDF Vocabulary Selection

### Core Vocabularies for Solid

| Vocabulary | Prefix | Use for |
|------------|--------|---------|
| FOAF | `foaf:` | Person, identity, social graph |
| Schema.org | `schema:` | Generic structured data (people, events, things) |
| LDP | `ldp:` | Containers, basic container membership |
| PIM Space | `pim:` | Storage, preferences, configuration |
| Solid Terms | `solid:` | OIDC issuer, type index, notifications |
| Dublin Core | `dc:` / `dcterms:` | Metadata (title, creator, date, description) |
| SKOS | `skos:` | Controlled vocabularies, concept schemes |
| OWL | `owl:` | Ontology relations, sameAs |
| RDFS | `rdfs:` | Labels, comments, subclass/subproperty |
| vCard | `vcard:` | Contact information |
| AS (ActivityStreams) | `as:` | Social activities, notifications |

### Vocabulary Lookup

Before creating custom predicates, search existing ontologies:

- **Linked Open Vocabularies (LOV)**: https://lov.linkeddata.es/
- **prefix.cc**: https://prefix.cc/ — namespace lookup
- **schema.org**: https://schema.org/ — widely adopted, search-engine friendly

---

## Shapes: ShEx and SHACL

Shapes define the expected structure of RDF data. They serve as schemas for validation and code generation.

### ShEx (Shape Expressions)

ShEx is the primary shape language used in the Solid ecosystem (e.g., by LDO).

**Basic ShEx shape:**

```shex
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX schema: <https://schema.org/>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

<PersonShape> {
  a foaf:Person ;
  foaf:name xsd:string ;
  foaf:mbox IRI? ;
  schema:birthDate xsd:date?
}
```

**Key concepts:**
- `{ }` — shape definition
- `?` — optional property
- `+` — one or more
- `*` — zero or more
- `IRI` — expects an IRI value
- `xsd:string`, `xsd:date` etc. — literal datatypes

### SHACL (Shapes Constraint Language)

SHACL is the W3C Recommendation for RDF validation, more expressive for complex constraints.

```turtle
@prefix sh: <http://www.w3.org/ns/shacl#>.
@prefix foaf: <http://xmlns.com/foaf/0.1/>.

<PersonShape>
    a sh:NodeShape ;
    sh:targetClass foaf:Person ;
    sh:property [
        sh:path foaf:name ;
        sh:datatype xsd:string ;
        sh:minCount 1 ;
    ] ;
    sh:property [
        sh:path foaf:mbox ;
        sh:nodeKind sh:IRI ;
        sh:maxCount 1 ;
    ] .
```

### When to Use Each

| | ShEx | SHACL |
|-|------|-------|
| Solid / LDO code generation | Preferred | Possible |
| W3C standard | Yes | Yes (Recommendation) |
| Validation complexity | Moderate | High (supports SPARQL constraints) |
| Tooling | LDO, shex.js | Apache Jena, TopBraid, pyshacl |

---

## MCP Adapters for Ontology Discovery

Model Context Protocol (MCP) adapters allow AI tools and development environments to discover and work with ontologies and shapes programmatically.

### W3C Data Shapes MCP Adapter

- PR: https://github.com/w3c/data-shapes/pull/566
- Provides: Access to SHACL/ShEx shape definitions via MCP
- Use case: AI-assisted shape authoring, validation, and discovery

### Discovering Ontologies via MCP

When building an MCP adapter for ontology discovery, key capabilities to expose:

1. **Search by term** — find classes and properties matching a keyword
2. **Namespace lookup** — resolve a prefix to its full IRI
3. **Shape retrieval** — fetch ShEx/SHACL shapes for a known type
4. **Type index traversal** — discover where a user stores instances of a given type on their pod

### Recommended Approach for Ontology Discovery in Apps

```
1. Check LOV (lov.linkeddata.es) for existing vocabulary terms
2. Use prefix.cc to resolve unknown prefixes
3. Use ShEx/SHACL shapes (from shapes repo or LDO) to validate data
4. For pod-specific discovery, traverse the user's type index
```

---

## Type Index Pattern

The type index is the standard Solid mechanism for linking RDF types to storage locations in a pod.

### Structure

```turtle
# Public type index at /settings/publicTypeIndex.ttl

@prefix solid: <http://www.w3.org/ns/solid/terms#>.
@prefix schema: <https://schema.org/>.

<#registration-contacts>
    a solid:TypeRegistration ;
    solid:forClass schema:Person ;
    solid:instanceContainer </contacts/> .

<#registration-notes>
    a solid:TypeRegistration ;
    solid:forClass schema:TextObject ;
    solid:instance </notes/main.ttl> .
```

### Two Registration Types

| Property | Meaning |
|----------|---------|
| `solid:instanceContainer` | All instances of this type live in this container |
| `solid:instance` | A single document containing instances of this type |

### Accessing the Type Index

1. Fetch the user's WebID profile
2. Find `solid:publicTypeIndex` and/or `solid:privateTypeIndex` predicates
3. Fetch the type index document
4. Query for `solid:TypeRegistration` entries with matching `solid:forClass`

---

## Data Modelling Checklist

When modelling a new data type for Solid:

- [ ] Check LOV for an existing vocabulary/class
- [ ] Define a ShEx shape for the type
- [ ] Add `rdf:type` statements to all instances
- [ ] Register the type in the user's type index
- [ ] Use stable, dereferenceable URIs for subjects
- [ ] Add `dcterms:created`, `dcterms:modified` metadata
- [ ] Document the shape in a public shapes repository
- [ ] Validate data against the shape before writing to the pod

---

## Resources

| Resource | URL |
|----------|-----|
| Linked Open Vocabularies | https://lov.linkeddata.es/ |
| prefix.cc | https://prefix.cc/ |
| schema.org | https://schema.org/ |
| W3C SHACL Spec | https://www.w3.org/TR/shacl/ |
| ShEx Spec | https://shex.io/ |
| W3C Data Shapes repo | https://github.com/w3c/data-shapes |
| Solid type indexes | https://solid.github.io/type-indexes/ |
| LDO (ShEx → TypeScript) | https://ldo.js.org/ |
