# next-skills

Opinionated agent skills for **Next.js 16 App Router** projects, distributed via the [`vercel-labs/skills`](https://github.com/vercel-labs/skills) CLI.

Each skill is a single `SKILL.md` with YAML frontmatter under `skills/<skill-name>/`. The skills are designed to be loaded by Claude Code and other compatible agents (OpenCode, Codex, Cursor, etc.).

## Skills in this repo

| Skill | Summary |
|---|---|
| [`nextjs-data-fetching`](skills/nextjs-data-fetching/SKILL.md) | Read with Server Components, mutate with Server Actions, never fetch via `useEffect`. Eight ❌→✅ pairs verified against the Next.js 16 docs, plus a rationalizations table and red-flag list. |

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

## Companion skill — strongly recommended

`nextjs-data-fetching` **never** recommends a raw `useEffect`. The React-side rule that backs that up — the five replacement patterns plus the `useMountEffect` escape hatch — lives in the **`no-use-effect`** skill (origin: [@alvinsng](https://x.com/alvinsng/status/2033969062834045089)). Install whichever published copy you trust alongside this one.

## Repo layout

```
next-skills/
├── README.md
└── skills/
    └── nextjs-data-fetching/
        └── SKILL.md
```

The `skills/` directory layout is one of the locations the `vercel-labs/skills` CLI auto-discovers, so `npx skills add lusentis/next-skills` finds every skill in this repo without extra config.

## Contributing

Each skill is one folder under `skills/` with a single `SKILL.md`. Frontmatter must include `name` and `description` (see [agentskills.io/specification](https://agentskills.io/specification)). PRs welcome.

## License

MIT
