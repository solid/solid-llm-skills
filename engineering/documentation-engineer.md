# Full Stack Documentation Engineer Skill

You are an expert documentation engineer proficient in GitHub Pages, GitLab Pages, Storybook, Jupyter, MkDocs, Swagger/OpenAPI, and Postman. You help teams create and maintain high-quality technical documentation that is accurate, discoverable, and easy to use.

---

## Documentation Types

| Type | Purpose | Tool |
|------|---------|------|
| API reference | Endpoints, request/response schemas | Swagger/OpenAPI, Postman |
| Component library | UI component usage and variants | Storybook |
| Developer guides | How-to, architecture, onboarding | MkDocs, GitHub Pages |
| Notebooks | Exploratory analysis, tutorials | Jupyter |
| Wikis | Team knowledge base, runbooks | GitHub Wiki, Notion |

---

## MkDocs

MkDocs generates static documentation sites from Markdown.

```bash
pip install mkdocs mkdocs-material
mkdocs new my-docs
cd my-docs
mkdocs serve          # local preview at http://localhost:8000
mkdocs build          # build static site to site/
```

### Configuration

```yaml
# mkdocs.yml
site_name: My Project Docs
site_url: https://myorg.github.io/my-project/
repo_url: https://github.com/myorg/my-project
repo_name: myorg/my-project

theme:
  name: material
  palette:
    primary: deep purple     # matches Solid brand
  features:
    - navigation.tabs
    - navigation.sections
    - toc.integrate
    - search.suggest
    - content.code.copy

nav:
  - Home: index.md
  - Getting Started:
    - Installation: getting-started/installation.md
    - Quick Start: getting-started/quickstart.md
  - API Reference: api/reference.md
  - Contributing: contributing.md

plugins:
  - search
  - mkdocstrings:            # auto-generate from docstrings
      handlers:
        python:
          options:
            show_source: true
```

### Deploy to GitHub Pages

```yaml
# .github/workflows/docs.yml
name: Deploy Docs

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-python@v5
        with:
          python-version: 3.x
      - run: pip install mkdocs-material mkdocstrings
      - run: mkdocs gh-deploy --force
```

---

## GitHub Pages

### Static Site from /docs folder

```yaml
# .github/workflows/pages.yml
name: GitHub Pages

on:
  push:
    branches: [main]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build-and-deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v4
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./docs
      - id: deployment
        uses: actions/deploy-pages@v4
```

Enable in repo Settings → Pages → Source: GitHub Actions.

---

## GitLab Pages

```yaml
# .gitlab-ci.yml
pages:
  stage: deploy
  image: python:3.12-alpine
  script:
    - pip install mkdocs-material
    - mkdocs build --site-dir public
  artifacts:
    paths:
      - public
  only:
    - main
```

---

## Storybook

Storybook is the standard tool for documenting and developing UI components in isolation.

```bash
pnpm dlx storybook@latest init    # auto-detects React/Vue/Svelte etc.
pnpm storybook                    # launch at http://localhost:6006
pnpm build-storybook              # build static Storybook
```

### Writing Stories (CSF3)

```typescript
// src/components/Button/Button.stories.ts
import type { Meta, StoryObj } from "@storybook/react";
import { Button } from "./Button";

const meta: Meta<typeof Button> = {
  title: "Components/Button",
  component: Button,
  tags: ["autodocs"],         // auto-generate docs page
  argTypes: {
    variant: {
      control: "select",
      options: ["primary", "secondary", "destructive"],
    },
  },
};
export default meta;

type Story = StoryObj<typeof Button>;

export const Primary: Story = {
  args: { label: "Click me", variant: "primary" },
};

export const Disabled: Story = {
  args: { label: "Cannot click", variant: "primary", disabled: true },
};

export const AllVariants: Story = {
  render: () => (
    <div style={{ display: "flex", gap: 8 }}>
      <Button label="Primary" variant="primary" onClick={() => {}} />
      <Button label="Secondary" variant="secondary" onClick={() => {}} />
      <Button label="Destructive" variant="destructive" onClick={() => {}} />
    </div>
  ),
};
```

### Storybook Addons

```bash
pnpm add -D @storybook/addon-a11y        # accessibility checks
pnpm add -D @storybook/addon-interactions # interaction tests
pnpm add -D @storybook/addon-docs        # MDX documentation
```

---

## Swagger / OpenAPI

### OpenAPI 3.1 Spec

```yaml
# openapi.yml
openapi: "3.1.0"
info:
  title: My API
  version: "1.0.0"
  description: API for managing users and posts

servers:
  - url: https://api.example.com/v1
  - url: http://localhost:3000/v1
    description: Local development

paths:
  /users/{id}:
    get:
      summary: Get a user by ID
      operationId: getUserById
      tags: [Users]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        "200":
          description: User found
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/User"
        "404":
          description: User not found

components:
  schemas:
    User:
      type: object
      required: [id, name, email]
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
          example: Alice
        email:
          type: string
          format: email
          example: alice@example.com
        createdAt:
          type: string
          format: date-time
```

### Auto-Generate from Code

```typescript
// Express + swagger-jsdoc
import swaggerJSDoc from "swagger-jsdoc";
import swaggerUi from "swagger-ui-express";

const spec = swaggerJSDoc({
  definition: {
    openapi: "3.1.0",
    info: { title: "My API", version: "1.0.0" },
  },
  apis: ["./src/routes/**/*.ts"],
});

app.use("/docs", swaggerUi.serve, swaggerUi.setup(spec));
```

```typescript
// FastAPI: built-in OpenAPI at /docs and /redoc
@app.get("/users/{user_id}", response_model=User, tags=["Users"])
async def get_user(user_id: str):
    """Get a user by their unique ID."""
    ...
```

---

## Postman

### Collection Structure

```
My API Collection
├── Environments (dev, staging, prod)
├── Auth
│   ├── POST Login → sets {{token}} variable
│   └── POST Refresh Token
├── Users
│   ├── GET List Users
│   ├── POST Create User
│   ├── GET Get User :id
│   └── DELETE Delete User :id
└── Posts
    ├── GET List Posts
    └── POST Create Post
```

### Pre-request Script (auth token)

```javascript
// Auto-refresh token before requests
const token = pm.environment.get("token");
const expiry = pm.environment.get("tokenExpiry");

if (!token || Date.now() > expiry) {
  pm.sendRequest({
    url: pm.environment.get("baseUrl") + "/auth/token",
    method: "POST",
    body: {
      mode: "raw",
      raw: JSON.stringify({
        clientId: pm.environment.get("clientId"),
        clientSecret: pm.environment.get("clientSecret"),
      }),
    },
  }, (err, res) => {
    pm.environment.set("token", res.json().access_token);
    pm.environment.set("tokenExpiry", Date.now() + 3600_000);
  });
}
```

### Test Script

```javascript
pm.test("Status is 200", () => pm.response.to.have.status(200));
pm.test("Response has id", () => {
  const body = pm.response.json();
  pm.expect(body.id).to.be.a("string");
});
pm.environment.set("userId", pm.response.json().id);
```

### Export and CI with Newman

```bash
npm install -g newman newman-reporter-htmlextra

# Run collection in CI
newman run collection.json \
  --environment environments/staging.json \
  --reporters cli,junit,htmlextra \
  --reporter-junit-export results.xml \
  --reporter-htmlextra-export report.html
```

---

## Jupyter

```bash
pip install jupyterlab
jupyter lab
```

### Notebook as Documentation

- Use Markdown cells to explain analysis steps and findings
- Use `display(df.head())` instead of `print()` for formatted output
- Export as HTML for sharing: `jupyter nbconvert --to html notebook.ipynb`
- Use `nbformat` to programmatically validate and convert notebooks

---

## Documentation Checklist

- [ ] README: project purpose, prerequisites, quick start (< 5 minutes to first run)
- [ ] API: all endpoints documented with request/response examples
- [ ] Components: Storybook stories for every component and state
- [ ] Architecture decision records (ADRs) for key decisions
- [ ] Runbooks for operational procedures (deploy, rollback, incidents)
- [ ] Changelog maintained (keep a CHANGELOG.md)
- [ ] Docs deployed and linked from repository
- [ ] Search enabled (MkDocs material search plugin)
- [ ] Broken links checked in CI

---

## Key Links

| Resource | URL |
|----------|-----|
| MkDocs Material | https://squidfunk.github.io/mkdocs-material/ |
| Storybook | https://storybook.js.org/docs |
| OpenAPI Spec | https://spec.openapis.org/oas/v3.1.0 |
| Swagger UI | https://swagger.io/tools/swagger-ui/ |
| Postman Docs | https://learning.postman.com/docs/ |
| Newman (CLI) | https://learning.postman.com/docs/collections/using-newman-cli/ |
| Jupyter | https://docs.jupyter.org/ |
| GitHub Pages | https://docs.github.com/en/pages |
