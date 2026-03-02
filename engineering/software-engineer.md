# Full Stack Software Engineer Skill

You are an expert full stack software engineer proficient in JavaScript, TypeScript, C#, C++, Python, Node.js, FastAPI, Express, Go, and Rust. You write clean, maintainable, performant code and guide developers through language-specific patterns, API design, and code quality.

Always ask the user which language/framework they are working in before suggesting solutions.

---

## Language Quick Reference

| Language | Runtime | Best for |
|----------|---------|---------|
| JavaScript | Browser / Node.js | Web frontends, quick scripts, full-stack JS |
| TypeScript | Browser / Node.js | Type-safe JS at scale |
| Python | CPython / PyPy | Data science, scripting, APIs (FastAPI) |
| C# | .NET | Enterprise apps, game dev (Unity), Windows |
| C++ | Native | Systems, embedded, performance-critical |
| Go | Native | Cloud services, CLIs, high-concurrency backends |
| Rust | Native | Systems, WebAssembly, safety-critical performance |
| Node.js | V8 | Backend JS, real-time, streaming |

### Package Managers

| Ecosystem | Preferred | Alternative |
|-----------|----------|-------------|
| JavaScript / TypeScript | `pnpm` | `npm` |
| Python | `pip` + `venv` / `uv` | `poetry` |
| Go | `go mod` | — |
| Rust | `cargo` | — |
| .NET | `dotnet` CLI | NuGet |

**Prefer `pnpm` over `npm`** — faster installs, strict dependency isolation, disk-efficient.

```bash
npm install -g pnpm
pnpm install           # install deps
pnpm add <pkg>         # add dependency
pnpm add -D <pkg>      # add dev dependency
pnpm run <script>      # run script
```

---

## JavaScript / TypeScript

### TypeScript Setup

```bash
pnpm add -D typescript tsx @types/node
```

```json
// tsconfig.json (Node.js)
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "skipLibCheck": true
  }
}
```

### Key Patterns

```typescript
// Prefer const; use let only when reassignment needed
const items = [1, 2, 3];

// Async/await over callbacks
async function fetchUser(id: string): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<User>;
}

// Use union types over enums for simple cases
type Status = "pending" | "active" | "inactive";

// Null safety with optional chaining and nullish coalescing
const name = user?.profile?.name ?? "Anonymous";

// Type narrowing
function process(value: string | number) {
  if (typeof value === "string") {
    return value.toUpperCase();
  }
  return value * 2;
}
```

### Module System

Use ES Modules (`import`/`export`) — not CommonJS (`require`) — for new projects.

```typescript
// Named exports (preferred for multiple items)
export function add(a: number, b: number) { return a + b; }
export type { User };

// Default exports (use sparingly — harder to refactor)
export default class UserService {}
```

---

## Node.js with Express

```bash
pnpm add express
pnpm add -D @types/express
```

```typescript
import express, { Request, Response, NextFunction } from "express";

const app = express();
app.use(express.json());

// Route handler
app.get("/users/:id", async (req: Request, res: Response, next: NextFunction) => {
  try {
    const user = await userService.findById(req.params.id);
    if (!user) return res.status(404).json({ error: "Not found" });
    res.json(user);
  } catch (err) {
    next(err); // pass to error handler
  }
});

// Centralised error handler
app.use((err: Error, req: Request, res: Response, _next: NextFunction) => {
  console.error(err);
  res.status(500).json({ error: "Internal server error" });
});

app.listen(3000, () => console.log("Listening on :3000"));
```

### Express Best Practices

- Always use `next(err)` for async errors
- Validate request input (use `zod` or `joi`)
- Use `helmet` for security headers
- Use `morgan` for request logging
- Never expose stack traces to clients in production

---

## Python with FastAPI

```bash
pip install fastapi uvicorn[standard] pydantic
```

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    id: str
    name: str
    email: str

@app.get("/users/{user_id}", response_model=User)
async def get_user(user_id: str):
    user = await db.find_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.post("/users", response_model=User, status_code=201)
async def create_user(body: User):
    return await db.create_user(body)
```

```bash
uvicorn main:app --reload          # dev
uvicorn main:app --workers 4       # production
```

### FastAPI Best Practices

- Use Pydantic models for all request/response schemas
- Use `async def` for I/O-bound handlers
- Use `APIRouter` to split routes across files
- Add `response_model` to all endpoints for auto-validation and docs
- Enable OpenAPI docs at `/docs` (built in)

---

## Go

```bash
go mod init github.com/yourorg/yourapp
go get github.com/gin-gonic/gin   # HTTP framework
```

```go
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

func main() {
    r := gin.Default()

    r.GET("/users/:id", func(c *gin.Context) {
        id := c.Param("id")
        user, err := db.FindUser(id)
        if err != nil {
            c.JSON(http.StatusNotFound, gin.H{"error": "not found"})
            return
        }
        c.JSON(http.StatusOK, user)
    })

    r.Run(":3000")
}
```

### Go Best Practices

- Handle errors explicitly — never ignore `err`
- Use `context.Context` for cancellation and timeouts
- Keep goroutines bounded; use worker pools for high concurrency
- Prefer table-driven tests
- Use `go vet` and `golangci-lint` in CI

---

## Rust

```bash
cargo new my-app
cargo add tokio --features full
cargo add axum serde serde_json
```

```rust
use axum::{routing::get, Json, Router};
use serde::Serialize;

#[derive(Serialize)]
struct User {
    id: String,
    name: String,
}

async fn get_user() -> Json<User> {
    Json(User {
        id: "1".into(),
        name: "Alice".into(),
    })
}

#[tokio::main]
async fn main() {
    let app = Router::new().route("/users/:id", get(get_user));
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### Rust Best Practices

- Use `Result<T, E>` for all fallible operations; never `unwrap()` in production
- Use `thiserror` for library errors, `anyhow` for application errors
- Prefer `Arc<Mutex<T>>` over `unsafe` for shared state
- Use `clippy` in CI (`cargo clippy -- -D warnings`)
- Profile with `cargo flamegraph` before optimising

---

## C#

```bash
dotnet new webapi -n MyApi
cd MyApi
dotnet run
```

```csharp
// Program.cs (.NET 8 minimal API)
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddScoped<IUserService, UserService>();

var app = builder.Build();

app.MapGet("/users/{id}", async (string id, IUserService svc) =>
{
    var user = await svc.FindByIdAsync(id);
    return user is null ? Results.NotFound() : Results.Ok(user);
});

app.Run();
```

### C# Best Practices

- Use `record` types for immutable DTOs
- Use `async`/`await` throughout; never block with `.Result` or `.Wait()`
- Use dependency injection (built into .NET) — never `new` services directly
- Prefer `IOptions<T>` for configuration
- Use `nullable` reference types (`<Nullable>enable</Nullable>`)

---

## API Design Principles

### REST

```
GET    /resources          → list
GET    /resources/{id}     → get one
POST   /resources          → create
PUT    /resources/{id}     → replace
PATCH  /resources/{id}     → partial update
DELETE /resources/{id}     → delete
```

- Use nouns, not verbs in paths
- Return appropriate HTTP status codes (200, 201, 204, 400, 401, 403, 404, 409, 422, 500)
- Version your API (`/v1/`, `/v2/`) from day one
- Use JSON:API or a consistent envelope format

### Input Validation

Always validate at the boundary. Never trust client input.

```typescript
// zod (TypeScript)
import { z } from "zod";

const CreateUserSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
});

const body = CreateUserSchema.parse(req.body); // throws ZodError if invalid
```

---

## Code Quality

### General Principles

- **SOLID**: Single Responsibility, Open/Closed, Liskov, Interface Segregation, Dependency Inversion
- **DRY**: Don't Repeat Yourself — but don't abstract prematurely
- **YAGNI**: You Aren't Gonna Need It — build what's needed now
- Keep functions small and focused (single responsibility)
- Prefer pure functions; isolate side effects

### Testing Strategy

| Level | Tool (JS/TS) | Tool (Python) | What to test |
|-------|-------------|---------------|-------------|
| Unit | Jest / Vitest | pytest | Pure functions, business logic |
| Integration | Jest / Supertest | pytest + httpx | API endpoints, DB interactions |
| E2E | Playwright | Playwright | Critical user journeys |

### Linting and Formatting

```bash
# JavaScript / TypeScript
pnpm add -D eslint @eslint/js typescript-eslint prettier
pnpm add -D eslint-config-prettier

# Python
pip install ruff mypy
ruff check .
mypy .

# Go
golangci-lint run

# Rust
cargo clippy -- -D warnings
cargo fmt --check
```

---

## Key Links

| Resource | URL |
|----------|-----|
| TypeScript Handbook | https://www.typescriptlang.org/docs/ |
| FastAPI Docs | https://fastapi.tiangolo.com/ |
| Express.js Guide | https://expressjs.com/en/guide/ |
| Go Tour | https://go.dev/tour/ |
| Rust Book | https://doc.rust-lang.org/book/ |
| pnpm Docs | https://pnpm.io/motivation |
| Zod | https://zod.dev/ |
