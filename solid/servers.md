# Solid Servers Skill

You are an expert on Solid server implementations — both public hosted servers and local development setups. Use this knowledge to help developers choose, configure, run, and debug Solid servers.

---

## Public Solid Servers

| Server | URL | Notes |
|--------|-----|-------|
| solidcommunity.net | https://solidcommunity.net | Powered by Pivot CSS; widely used community server |
| solidweb.org | https://solidweb.org | Community-hosted |
| start.inrupt.com | https://start.inrupt.com | Inrupt's hosted pod service (ESS) |

### Choosing a Public Server or allow interoperability for all servers. Ask the user

- **solidcommunity.net** — best for open community apps; uses older DPoP-based OIDC (Pivot flavour)
- **solidweb.org** — alternative community option
- **start.inrupt.com (ESS)** — enterprise-grade, uses Inrupt's Enterprise Solid Server; best compatibility with Inrupt SDK

---

## Community Solid Server (CSS)

**Repo**: https://github.com/CommunitySolidServer/CommunitySolidServer
**Docs**: https://communitysolidserver.github.io/CommunitySolidServer/latest/

The reference open-source implementation of the Solid specification. Highly modular and configurable.

### Requirements

- Node.js 18+
- npm or npx

### Quick Start (in-memory, ephemeral)

```bash
npx @solid/community-server
```

Server available at http://localhost:3000/

### Persistent File-Based Storage

```bash
npx @solid/community-server -c @css:config/file.json -f ./data
```

Data stored in `./data/` directory.

### Global Install

```bash
npm install -g @solid/community-server
community-solid-server -c @css:config/file.json -f ./data
```

### Key CLI Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--port, -p` | `3000` | TCP port to listen on |
| `--baseUrl, -b` | `http://localhost:3000/` | Base URL used in generated resource URIs |
| `--config, -c` | `@css:config/default.json` | Configuration preset |
| `--rootFilePath, -f` | `./` | Directory for file-based storage |
| `--loggingLevel, -l` | `info` | Log level (`debug`, `info`, `warn`, `error`) |
| `--workers, -w` | `1` | Number of worker threads |
| `--sparqlEndpoint` | — | SPARQL endpoint URL for SPARQL-backed storage |

### Environment Variables

All flags can be set as environment variables using the `CSS_` prefix:

```bash
CSS_PORT=4000
CSS_LOGGING_LEVEL=debug
CSS_ROOT_FILE_PATH=/var/solid/data
```

### Storage Backends

| Config | Backend | Use case |
|--------|---------|---------|
| `@css:config/default.json` | In-memory | Quick testing, no persistence |
| `@css:config/file.json` | File system | Local dev with persistence |
| `--sparqlEndpoint` | SPARQL | Production or graph DB backends |

### Docker

```bash
# Run with file-based storage mounted at ~/Solid
docker run --rm \
  -v ~/Solid:/data \
  -p 3000:3000 \
  solidproject/community-server:latest \
  -c @css:config/file.json -f /data
```

### Kubernetes

Use the official Helm chart: https://github.com/CommunitySolidServer/css-helm-chart

### Build from Source

```bash
git clone https://github.com/CommunitySolidServer/CommunitySolidServer
cd CommunitySolidServer
npm ci
npm start
```

### Access Control

CSS supports both **WAC** (Web Access Control) and **ACP** (Access Control Policy) depending on configuration. Default config uses WAC.

---

## Pivot CSS

**Repo**: https://github.com/solid-contrib/pivot
**Used by**: solidcommunity.net

Pivot is a spec-compliant Solid server built as a thin wrapper/remix of CSS. It takes a different philosophy on consent and access control.

### Key Differences from CSS

| Aspect | CSS | Pivot |
|--------|-----|-------|
| OIDC version | Latest Solid-OIDC (UMA) | Older DPoP-based OIDC |
| Access control model | WAC (default) or ACP | User-centric, DPoP |
| Auth approach | Standard | App gets full pod access when authenticated |
| Focus | Reference implementation | Simplified consent management |

### When to Use Pivot

- Building apps that target **solidcommunity.net** users
- When you need the exact server behaviour of solidcommunity.net locally
- Testing compatibility with the older Solid-OIDC DPoP flow

### Running Pivot Locally

```bash
git clone https://github.com/solid-contrib/pivot
cd pivot
npm install
npm start
```

Default port: 3000 (same as CSS)

---

## Local Development Tips

### Running Two Servers Side-by-Side

To test cross-server behaviour (e.g., Alice on CSS, Bob on Pivot):

```bash
# CSS on port 3000
npx @solid/community-server -p 3000 -c @css:config/file.json -f ./data-css

# Pivot on port 3001
cd pivot && PORT=3001 npm start
```

### Setting baseUrl for Deployed Environments

When deploying behind a reverse proxy (Nginx, Caddy):

```bash
npx @solid/community-server \
  -b https://solid.example.com/ \
  -c @css:config/file.json \
  -f /var/solid/data
```

The `--baseUrl` must match the public-facing URL exactly — resources are minted using this value.

### Debugging

```bash
npx @solid/community-server -l debug
```

Or via environment variable:

```bash
CSS_LOGGING_LEVEL=debug npx @solid/community-server
```

### Creating a Test Account (CSS)

Navigate to http://localhost:3000/ — CSS provides a UI for pod registration when running locally.

---

## Conformance Testing

Use the Solid conformance test harness to verify your server implementation:

- Repo: https://github.com/solid-contrib/conformance-test-harness
- Inrupt test suite info: https://www.inrupt.com/blog/conformance-test-suite

```bash
git clone https://github.com/solid-contrib/conformance-test-harness
cd conformance-test-harness
# See repo README for configuration and execution steps
```

---

## Quick Reference

| Task | Command |
|------|---------|
| Start CSS (ephemeral) | `npx @solid/community-server` |
| Start CSS (persistent) | `npx @solid/community-server -c @css:config/file.json -f ./data` |
| Start CSS (custom port) | `npx @solid/community-server -p 4000` |
| Start CSS (Docker) | `docker run --rm -v ~/Solid:/data -p 3000:3000 solidproject/community-server:latest -c @css:config/file.json -f /data` |
| Debug logging | `npx @solid/community-server -l debug` |
