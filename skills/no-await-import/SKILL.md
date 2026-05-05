---
name: no-await-import
description: Use when writing, reviewing, or refactoring JavaScript or TypeScript and you are about to use `await import(...)`, dynamic `import(...)`, computed module paths, conditional imports, lazy-loading imports, or test-time module isolation. Forbids `await import()` completely in production code. Allows it only rarely in tests, and only with an inline comment explaining why it is the best option and why static imports or framework-native lazy loading cannot work.
---

# No `await import()` — static imports only

## The Rule

**Never use `await import()` or dynamic `import(...)` in application, library, build, script, or runtime code.**

Default to top-level static imports:

```ts
import { createClient } from "@/lib/db/client";
import { renderInvoice } from "@/lib/invoices/render";
```

Do not write:

```ts
const { createClient } = await import("@/lib/db/client");
```

This is a hard ban. It applies even when the import path is a literal string and even when the code is inside an async function.

## Why

Dynamic imports hide dependencies from readers and tooling. They make module graphs harder to audit, weaken type/navigation support, create async execution edges in places that should be synchronous, and often disguise architecture problems as "lazy loading."

Most claimed reasons for `await import()` have better answers:

| Claim | Use instead |
|---|---|
| "I only need it in this branch" | Static import at the top. Let bundlers tree-shake and split using the framework's supported mechanisms. |
| "I want to avoid a circular dependency" | Fix the module boundary. Move shared types/constants/helpers to a lower-level module. |
| "I need lazy UI loading" | Use the framework-native API: `next/dynamic`, `React.lazy`, route-level splitting, or the local pattern already in the repo. |
| "This package is optional" | Create an explicit adapter boundary and document the optional dependency there. Do not hide it with opportunistic imports. |
| "This only runs in Node" | Use static imports in a Node-only file, or split browser/server entrypoints explicitly. |
| "It avoids startup cost" | Measure first. If startup cost is real, use a documented lazy-loading primitive with ownership and tests. |

## What to do when you see one

1. Replace literal dynamic imports with top-level static imports.
2. If the import was conditional because of environment boundaries, split the module into explicit server/client, node/browser, or test/production files.
3. If the import was for UI code splitting, use the project's established lazy-loading primitive instead of raw `import()`.
4. If the import path is computed, replace the runtime module lookup with an explicit registry.

Example registry:

```ts
import { stripeProvider } from "./providers/stripe";
import { paddleProvider } from "./providers/paddle";

const providers = {
  stripe: stripeProvider,
  paddle: paddleProvider,
} as const;

export function getProvider(name: keyof typeof providers) {
  return providers[name];
}
```

## Test-only exception

`await import()` is allowed **only rarely in tests** when a test runner genuinely requires a fresh module evaluation after setting mocks, globals, environment variables, or fake timers.

Every exception must satisfy all of these conditions:

- The file is clearly test-only: `*.test.*`, `*.spec.*`, `__tests__/`, test fixtures, or test utilities.
- Static imports, dependency injection, and the test runner's normal mock APIs cannot express the case cleanly.
- The dynamic import is local to the specific test that needs fresh module evaluation.
- The line immediately above it contains a comment explaining why `await import()` is the best option.

Allowed shape:

```ts
process.env.FEATURE_FLAG = "enabled";
vi.resetModules();

// await import is required here so this test evaluates config after FEATURE_FLAG is set; static import would capture the previous environment.
const { getConfig } = await import("../config");
```

If you cannot write that comment truthfully, do not use `await import()`.

## Review checklist

Flag all of these:

- `await import("...")`
- `import("...").then(...)`
- `const mod = import("...")`
- computed module paths such as `import("./providers/" + name)`
- dynamic imports used to dodge circular dependencies
- dynamic imports used in production code for tests, mocks, or environment setup

In production code, the fix is not "add a comment." The fix is to remove the dynamic import.
