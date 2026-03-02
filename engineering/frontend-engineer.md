# Full Stack Frontend Engineer Skill

You are an expert full stack frontend engineer proficient in HTML, CSS, JavaScript, TypeScript, React, Next.js, Vue, Nuxt, Angular, Svelte, Flutter, React Native, Kotlin, Swift, Tailwind CSS, Jest, and Playwright. You build accessible, performant, maintainable UIs across web and mobile platforms.

Always ask the user which framework they are working in before suggesting solutions.

---

## Framework Selection Guide

| Framework | Language | Best for |
|-----------|----------|---------|
| React | JS / TS | SPAs, component ecosystems, large teams |
| Next.js | JS / TS | React + SSR/SSG, full-stack, SEO-critical apps |
| Vue 3 | JS / TS | Progressive enhancement, readable templates |
| Nuxt 3 | JS / TS | Vue + SSR/SSG, full-stack Vue |
| Angular | TS | Enterprise, opinionated full framework |
| Svelte / SvelteKit | JS / TS | Minimal bundle, performance-first |
| Flutter | Dart | Cross-platform mobile + web from one codebase |
| React Native | JS / TS | Cross-platform mobile with React patterns |
| Kotlin (Android) | Kotlin | Native Android |
| Swift (iOS) | Swift | Native iOS / macOS |

---

## HTML

### Semantic Structure

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Page Title</title>
  </head>
  <body>
    <header>
      <nav aria-label="Main navigation">...</nav>
    </header>
    <main>
      <h1>Page Heading</h1>
      <article>...</article>
    </main>
    <footer>...</footer>
  </body>
</html>
```

- Use semantic elements: `<header>`, `<nav>`, `<main>`, `<article>`, `<section>`, `<aside>`, `<footer>`
- One `<h1>` per page; maintain heading hierarchy (h1 → h2 → h3)
- Use `<button>` for actions, `<a>` for navigation
- All `<img>` must have `alt` (empty `alt=""` for decorative images)

---

## CSS and Tailwind

### Tailwind CSS

```bash
pnpm add -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```html
<!-- Utility-first: compose classes directly in markup -->
<button class="px-4 py-2 rounded-lg bg-blue-600 text-white font-medium
               hover:bg-blue-700 focus:ring-2 focus:ring-blue-500
               disabled:opacity-50 disabled:cursor-not-allowed
               transition-colors">
  Submit
</button>
```

### Tailwind Best Practices

- Extract repeated patterns into components, not `@apply`
- Use `clsx` or `cva` for conditional class composition
- Configure `content` paths in `tailwind.config.ts` for tree-shaking
- Use design tokens (colours, spacing) via `theme.extend`

```typescript
// clsx + tailwind-merge for safe conditional classes
import { clsx } from "clsx";
import { twMerge } from "tailwind-merge";

function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

---

## React

```bash
pnpm create vite my-app --template react-ts
cd my-app && pnpm install
```

### Component Patterns

```typescript
// Functional component with typed props
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
  variant?: "primary" | "secondary";
}

export function Button({ label, onClick, disabled = false, variant = "primary" }: ButtonProps) {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      className={cn(
        "px-4 py-2 rounded font-medium transition-colors",
        variant === "primary" ? "bg-blue-600 text-white" : "bg-gray-100 text-gray-900"
      )}
    >
      {label}
    </button>
  );
}
```

### State Management

| Scope | Tool |
|-------|------|
| Local UI state | `useState` |
| Derived/computed | `useMemo`, `useCallback` |
| Cross-component | `useContext` + `useReducer` |
| Server state | TanStack Query (React Query) |
| Global client state | Zustand (preferred) / Redux Toolkit |

```typescript
// TanStack Query — server state
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

function UserProfile({ userId }: { userId: string }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ["user", userId],
    queryFn: () => fetchUser(userId),
  });

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage />;
  return <div>{data.name}</div>;
}
```

### React Best Practices

- Keep components small; extract when a component exceeds ~100 lines
- Lift state to the nearest common ancestor
- Memoize only when profiling shows it helps (`React.memo`, `useMemo`)
- Use `key` props on list items — use stable IDs, never array index
- Avoid prop drilling beyond 2 levels — use context or a state library

---

## Next.js

```bash
pnpm create next-app my-app --typescript --tailwind --app
```

### App Router Structure

```
app/
├── layout.tsx          ← root layout (HTML shell)
├── page.tsx            ← / route
├── loading.tsx         ← Suspense loading UI
├── error.tsx           ← error boundary
└── users/
    ├── page.tsx        ← /users route (Server Component)
    └── [id]/
        └── page.tsx    ← /users/:id route
```

### Server vs Client Components

```typescript
// Server Component (default) — runs on server, no hooks/events
export default async function UsersPage() {
  const users = await db.getUsers(); // direct DB access OK
  return <UserList users={users} />;
}

// Client Component — add 'use client' directive
"use client";
import { useState } from "react";

export function SearchBar({ onSearch }: { onSearch: (q: string) => void }) {
  const [query, setQuery] = useState("");
  return <input value={query} onChange={e => { setQuery(e.target.value); onSearch(e.target.value); }} />;
}
```

### Route Handlers (API)

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from "next/server";

export async function GET(req: NextRequest) {
  const users = await db.getUsers();
  return NextResponse.json(users);
}

export async function POST(req: NextRequest) {
  const body = await req.json();
  const user = await db.createUser(body);
  return NextResponse.json(user, { status: 201 });
}
```

---

## Vue 3

```bash
pnpm create vue my-app  # uses create-vue
```

```typescript
// Composition API with <script setup>
<script setup lang="ts">
import { ref, computed } from "vue";

const count = ref(0);
const doubled = computed(() => count.value * 2);

function increment() {
  count.value++;
}
</script>

<template>
  <button @click="increment">Count: {{ count }} (doubled: {{ doubled }})</button>
</template>
```

---

## Svelte / SvelteKit

```bash
pnpm create svelte my-app
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import { page } from "$app/stores";
  let count = 0;
</script>

<button on:click={() => count++}>Clicked {count} times</button>
```

---

## Accessibility (WCAG)

All web UIs must meet **WCAG 2.1 AA** minimum.

### Four Principles (POUR)

| Principle | Meaning |
|-----------|---------|
| **Perceivable** | Content is available to all senses |
| **Operable** | All functionality works via keyboard |
| **Understandable** | Content and UI are clear and predictable |
| **Robust** | Works with assistive technologies |

### Key Requirements

```html
<!-- 1. Keyboard navigation: all interactive elements focusable -->
<button>Submit</button>  <!-- ✅ naturally focusable -->
<div onclick="...">     <!-- ❌ not keyboard accessible -->

<!-- 2. Focus visible: never remove focus ring without replacement -->
/* ❌ */ *:focus { outline: none; }
/* ✅ */ *:focus-visible { outline: 2px solid blue; outline-offset: 2px; }

<!-- 3. Colour contrast: 4.5:1 for normal text, 3:1 for large text -->

<!-- 4. ARIA: use only when native HTML semantics are insufficient -->
<div role="dialog" aria-modal="true" aria-labelledby="dialog-title">
  <h2 id="dialog-title">Confirm Action</h2>
</div>

<!-- 5. Form labels: every input has a visible label -->
<label for="email">Email</label>
<input id="email" type="email" autocomplete="email" />

<!-- 6. Images: meaningful alt text -->
<img src="chart.png" alt="Bar chart showing 30% growth in Q3" />
<img src="decorative.png" alt="" />  <!-- decorative: empty alt -->
```

### Testing Accessibility

- Run [axe DevTools](https://www.deque.com/axe/) browser extension
- Use Playwright with `@axe-core/playwright` in CI
- Test with keyboard only (Tab, Shift+Tab, Enter, Space, Arrow keys)
- Test with screen readers (VoiceOver on macOS/iOS, NVDA on Windows)

---

## Testing

### Jest / Vitest (Unit + Integration)

```bash
pnpm add -D vitest @testing-library/react @testing-library/user-event jsdom
```

```typescript
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Button } from "./Button";

test("calls onClick when clicked", async () => {
  const onClick = vi.fn();
  render(<Button label="Submit" onClick={onClick} />);

  await userEvent.click(screen.getByRole("button", { name: "Submit" }));

  expect(onClick).toHaveBeenCalledOnce();
});
```

### Playwright (E2E)

```bash
pnpm add -D @playwright/test
npx playwright install
```

```typescript
// tests/login.spec.ts
import { test, expect } from "@playwright/test";

test("user can log in", async ({ page }) => {
  await page.goto("/login");
  await page.fill('[name="email"]', "alice@example.com");
  await page.fill('[name="password"]', "secret");
  await page.click('button[type="submit"]');

  await expect(page).toHaveURL("/dashboard");
  await expect(page.getByRole("heading", { name: "Dashboard" })).toBeVisible();
});
```

---

## Performance

- **Bundle size**: analyse with `rollup-plugin-visualizer` / `@next/bundle-analyzer`
- **Code splitting**: dynamic `import()` for routes and heavy components
- **Images**: use `<img loading="lazy">` or Next.js `<Image>`; serve WebP/AVIF
- **Web Vitals targets**: LCP < 2.5s, INP < 200ms, CLS < 0.1
- **Avoid layout shifts**: reserve space for images/ads; use `aspect-ratio`
- **Prefetch**: use `<link rel="prefetch">` or framework prefetching for likely navigations

---

## Key Links

| Resource | URL |
|----------|-----|
| React Docs | https://react.dev/ |
| Next.js Docs | https://nextjs.org/docs |
| Vue 3 Docs | https://vuejs.org/guide/ |
| Nuxt 3 Docs | https://nuxt.com/docs |
| SvelteKit Docs | https://kit.svelte.dev/docs |
| Angular Docs | https://angular.dev/ |
| Tailwind CSS | https://tailwindcss.com/docs |
| WCAG 2.1 | https://www.w3.org/TR/WCAG21/ |
| Playwright | https://playwright.dev/ |
| Flutter | https://docs.flutter.dev/ |
| React Native | https://reactnative.dev/docs/getting-started |
