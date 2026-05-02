# next-skills

Opinionated agent skills for **Next.js 16 App Router** projects, distributed via the [`vercel-labs/skills`](https://github.com/vercel-labs/skills) CLI.

Each skill is a single `SKILL.md` with YAML frontmatter under `skills/<skill-name>/`. The skills are designed to be loaded by Claude Code and other compatible agents (OpenCode, Codex, Cursor, etc.).

## Skills in this repo

| Skill | Summary |
|---|---|
| [`nextjs-data-fetching`](skills/nextjs-data-fetching/SKILL.md) | Read with Server Components, mutate with Server Actions, never fetch via `useEffect`. Eight ❌→✅ pairs verified against the Next.js 16 docs, plus a rationalizations table and red-flag list. |
| [`nextjs-forms`](skills/nextjs-forms/SKILL.md) | One shared toolkit on TanStack Form + Zod + shadcn `Field` for every form in the app. Every form has an explicit Save button gated by dirty + valid state — no auto-save, no save-on-blur. Bans `useState` for field values, `react-hook-form`, hand-rolled dirty tracking, inline `toast.success`/`toast.error`. Pairs with the dedicated `tanstack-form` skill. |
| [`nextjs-usestate`](skills/nextjs-usestate/SKILL.md) | `useState` is for ephemeral, local, view-only UI state — almost never the right answer in an App Router app. Thirteen ❌→✅ pairs covering URL state, server data, form values, derived state, prop mirroring, cookies for cross-render preferences, refs for non-render values, and the big one: open/closed state for shadcn / Radix / Base UI primitives belongs to the primitive, not to `useState`. **Explicitly forbids** Zustand, Jotai, Redux, Redux Toolkit, Recoil, Valtio, and MobX — every problem they solve in a classic SPA is solved better by App Router primitives (URL `searchParams`, cookies, Server Components, `useOptimistic`). |

More skills will be added over time.

## Install

Requires the [`vercel-labs/skills`](https://github.com/vercel-labs/skills) CLI (no global install — runs via `npx`).

### Install everything in this repo

```bash
npx skills add lusentis/next-skills -a claude-code
```

### Install a specific skill

```bash
npx skills add lusentis/next-skills --skill nextjs-data-fetching -a claude-code
npx skills add lusentis/next-skills --skill nextjs-forms -a claude-code
```

### List available skills before installing

```bash
npx skills add lusentis/next-skills --list
```

### Other agents

`-a claude-code` targets Claude Code. Drop the flag (or substitute another agent identifier — `opencode`, `codex`, `cursor`, etc.) to install elsewhere. The CLI also supports `-g` for global vs project-local installs and `-y` for non-interactive CI use.

## What `nextjs-data-fetching` enforces

The headline rule: **read with Server Components, mutate with Server Actions, never fetch via `useEffect`.**

It bans the most common modern-Next.js anti-pattern — calling a Server Action from a Client Component (often inside `useEffect`) just to load data — and gives explicit ✅/❌ replacement code for the eight cases the rule actually shows up in:

| # | ❌ Anti-pattern | ✅ Solution |
|---|---|---|
| 1 | Reading data via a Server Action in `useEffect` | Async Server Component (`await service()` directly) |
| 2 | Filter/tab state in `useState` + Server Action refetch | URL `searchParams` + Server Component re-render |
| 3 | Manual `setState` after a mutation | `revalidatePath` / `revalidateTag` / `refresh` inside the action |
| 4 | `getX` / `listX` files under `lib/actions/` | Reads live in `lib/services/`, called direct from Server Component |
| 5 | Awaiting fetch in parent before passing to Client child | Pass `Promise<T>` + `use()` + `<Suspense>` for streaming |
| 6 | `useState` + Server Action read-back for optimistic UI | `useOptimistic` |
| 7 | Polling / `setInterval` + Server Action | `GET` Route Handler + SWR / React Query |
| 8 | Whole page set to `"use client"` for one piece of state | Keep page as Server Component; smallest leaf gets `"use client"` |

It also ships a scope check (refuses to apply outside Next.js 16 App Router), a 16-row rationalizations table, and a red-flag list for code review.

## What `nextjs-forms` enforces

The headline rule: **every form goes through one shared toolkit on TanStack Form + Zod + the shadcn `Field` primitive, and every form has an explicit Save button gated by dirty + valid state — no auto-save, no save-on-blur.**

It bans hand-rolling the same five concerns in every form (field state in `useState`, raw `useForm` from `@tanstack/react-form`, custom dirty tracking, inline `toast.success`/`toast.error`, ad-hoc 422 error mapping) and codifies two hooks — `useEditForm` and `useCreateForm` — that own them once.

Prereqs the skill checks before applying: Next.js 16 App Router, `@tanstack/react-form` in deps, Zod v4, shadcn/ui configured (registry-agnostic — Radix-based or Base UI-based both work), the required shadcn components installed (`field`, `input`, `textarea`, `select`, `checkbox`, `switch`, `radio-group`, `button`, `label`, `sonner`), and a `lib/forms/` toolkit exporting the shared surface.

## Companion skills — strongly recommended

`nextjs-data-fetching` **never** recommends a raw `useEffect`. The React-side rule that backs that up — the five replacement patterns plus the `useMountEffect` escape hatch — lives in the **`no-use-effect`** skill (origin: [@alvinsng](https://x.com/alvinsng/status/2033969062834045089), published as part of [`factory-ai/factory-plugins`](https://github.com/factory-ai/factory-plugins)).

`nextjs-forms` governs *where* TanStack Form fits in a Next.js + shadcn project, but doesn't teach the library itself. For idiomatic TanStack Form usage (`form.Field`, `form.Subscribe`, validators, field arrays, async validation, the full API surface), pair it with the dedicated **`tanstack-form`** skill from [`tanstack-skills/tanstack-skills`](https://github.com/tanstack-skills/tanstack-skills).

Install everything together:

```bash
npx skills add https://github.com/factory-ai/factory-plugins --skill no-use-effect -a claude-code
npx skills add https://github.com/tanstack-skills/tanstack-skills --skill tanstack-form -a claude-code
npx skills add lusentis/next-skills -a claude-code
```

## Repo layout

```
next-skills/
├── README.md
└── skills/
    ├── nextjs-data-fetching/
    │   └── SKILL.md
    └── nextjs-forms/
        └── SKILL.md
```

The `skills/` directory layout is one of the locations the `vercel-labs/skills` CLI auto-discovers, so `npx skills add lusentis/next-skills` finds every skill in this repo without extra config.

## Contributing

Each skill is one folder under `skills/` with a single `SKILL.md`. Frontmatter must include `name` and `description` (see [agentskills.io/specification](https://agentskills.io/specification)). PRs welcome.

## License

MIT
