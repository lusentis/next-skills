---
name: nextjs-forms
description: Use when building or editing any form in a Next.js 16 App Router app — create dialogs, edit panels, settings UIs, anything with input fields that persist to a backend. Triggers on words "form", "edit panel", "create dialog", "settings", "wire up to a service", "save button", "dirty state". Triggers on symptoms reaching for `useState` to hold field values or submit state, importing `useForm` from `@tanstack/react-form` (raw, ungoverned), importing `react-hook-form`, hand-rolling dirty-state tracking, hand-rolling `toast.success`/`toast.error` on submit, components with `<input>`, `<textarea>`, `<select>`, `<Checkbox>`, or `<Switch>` bound to `useState`.
---

# Next.js Forms — TanStack Form + Zod + shadcn `Field`

## Scope

Applies to **Next.js 16 App Router** projects that have **shadcn/ui** configured. The skill is registry-agnostic: works with both the Radix-based shadcn registry and the Base UI-based registry. Picks no side between them — uses only the components that exist in both.

If the project is on Pages Router, an older Next.js, or has no shadcn/ui setup, refuse to apply and say so. If the project standardizes on `react-hook-form` (or any other form library) and won't migrate, also refuse — mixing form libraries inside one app is a non-starter.

## Companion skill — strongly recommended

This skill governs *where* TanStack Form fits in a Next.js + shadcn project (which hooks, which patterns, which red flags). It does **not** teach TanStack Form itself. For idiomatic TanStack Form usage — `form.Field`, `form.Subscribe`, validators, field arrays, async validation, the full API surface — install the dedicated companion skill alongside this one:

```bash
npx skills add https://github.com/tanstack-skills/tanstack-skills --skill tanstack-form
npx skills add lusentis/next-skills --skill nextjs-forms
```

If the agent reaches for a TanStack Form API you're unsure about, that companion skill is the source of truth. This skill defers to it for everything below the `useEditForm` / `useCreateForm` boundary.

## The Rule

**Every form goes through one shared toolkit built on TanStack Form + Zod. Every form has an explicit Save button gated by dirty + valid state. No auto-save, no save-on-blur, no debounce. Never `useState` for field values. Never raw `useForm` from `@tanstack/react-form`. Never inline-toast on save.**

A "form" is any UI with `<input>` / `<textarea>` / `<select>` / `<Checkbox>` / `<Switch>` / `<RadioGroup>` whose value persists to a backend (POST / PATCH / PUT). Display-only fields, search boxes that only filter, and fire-and-forget toggles that take no field state are not forms.

## Prerequisites checklist

Before writing any form code, verify:

1. **Next.js 16 App Router** — `package.json` has `"next": "^16…"` and routes live under `app/`.
2. **TanStack Form** — `@tanstack/react-form` is in `dependencies`. If `react-hook-form` is also present, stop and ask the user which one wins; do not silently mix.
3. **Zod v4** — `zod` >= 4 (import paths use `zod/v4`).
4. **shadcn/ui configured** — `components.json` exists at the project root. Either Radix-based or Base UI-based registry is fine.
5. **Required shadcn components installed** — at minimum:
   - `field` (provides `Field`, `FieldLabel`, `FieldDescription`, `FieldError`)
   - `input`, `textarea`, `select`, `checkbox`, `switch`, `radio-group`, `button`, `label`
   - `sonner` — and `<Toaster richColors position="top-right" />` mounted in `app/layout.tsx`
6. **Forms toolkit exists** — `lib/forms/index.ts` exports `useEditForm`, `useCreateForm`, `FormProvider`, `useFormContext`, `FormField`, `FormActions`, `mapFormError`. If it doesn't exist yet, build it once using the reference implementation in the [Toolkit reference](#toolkit-reference) section below — every form in the codebase shares it.

If any of items 1–5 is missing, fix the prereq before writing form code. If item 6 is missing, build the toolkit first; do not write a one-off form that bypasses it.

## Two patterns, one toolkit

| Form kind | Hook | Submit trigger | After success |
|---|---|---|---|
| **Edit** (panel, settings, inline editor) | `useEditForm` | `<Button type="submit">Save</Button>` — disabled unless `isDirty && canSubmit` | Form stays open. Hook resets defaults to the saved value so `isDirty` returns to `false`. |
| **Create** (dialog, new-record form) | `useCreateForm` | `<Button type="submit">Create</Button>` — disabled unless `canSubmit` | `onSuccess` callback fires (close dialog, route, etc.). |

Both share the `<FormProvider>` + `<FormField>` rendering layer, both use the same Zod schema as a validator, both route errors through `mapFormError`, both gate the Save/Create button via `form.Subscribe`. The differences are: edit forms additionally gate on `isDirty` and reset their baseline after a successful save; create forms close.

## Edit form pattern (manual save, dirty-gated)

```tsx
"use client";

import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import {
  FormActions,
  FormField,
  FormProvider,
  useEditForm,
} from "@/lib/forms";
import { projectsService } from "@/lib/services/projects.service";
import { type Project, ProjectSchema } from "@/lib/types/project";

export function ProjectEditPanel({ project }: { project: Project }) {
  const form = useEditForm<Project>({
    schema: ProjectSchema,
    defaultValues: project,
    save: (value, { signal }) =>
      projectsService.update(project.id, value, { signal }),
  });

  return (
    <FormProvider form={form as never}>
      <form
        onSubmit={(e) => {
          e.preventDefault();
          void form.handleSubmit();
        }}
      >
        <FormField name="name" label="Name">
          {(field) => (
            <Input
              id={String(field.name)}
              value={String(field.state.value ?? "")}
              onChange={(e) => field.handleChange(e.target.value)}
              onBlur={field.handleBlur}
              aria-invalid={
                field.state.meta.isTouched && !field.state.meta.isValid
              }
            />
          )}
        </FormField>

        <FormField name="description" label="Description">
          {(field) => (
            <Textarea
              id={String(field.name)}
              value={String(field.state.value ?? "")}
              onChange={(e) => field.handleChange(e.target.value)}
              onBlur={field.handleBlur}
              aria-invalid={
                field.state.meta.isTouched && !field.state.meta.isValid
              }
            />
          )}
        </FormField>

        <FormActions
          submitLabel="Save"
          submittingLabel="Saving…"
          resetLabel="Reset"
        />
      </form>
    </FormProvider>
  );
}
```

What the hook owns (and you must not reimplement):

- **Dirty tracking**: drives `isDirty` from a deep-equal compare of current values vs the current baseline (initial `defaultValues` on mount, then the most-recently-saved value).
- **Submit gating**: exposes `canSubmit = isValid && isDirty && !isSubmitting`. `<FormActions>` reads it via `form.Subscribe`.
- **Baseline reset on success**: after the `save` callback resolves, calls `form.reset(savedValue)` so the form's new "clean" state is the saved value. Editing back to the just-saved value should leave `isDirty === false`.
- **AbortController** on submit — if the user re-submits while a save is in flight, the in-flight one is aborted. (In practice this should be rare since the button is disabled while submitting.)
- **Error mapping**: thrown errors flow through `mapFormError` (see [Error handling](#error-handling)). The `save` callback **must throw on failure** — never catch.
- **Optional success toast** — if your `mapFormError` flavor emits success toasts on edit, configure it once globally; do not toast from form components.

What the consumer owns:

- The `<form onSubmit>` wrapper that calls `form.handleSubmit()`.
- Wiring `aria-invalid` on each input: `field.state.meta.isTouched && !field.state.meta.isValid`. `<FormField>` only sets `data-invalid` on the wrapper.
- Passing `{ signal }` straight through to the service method.

### Reset / Cancel

`<FormActions>` ships a Reset button that calls `form.reset()` (back to the current baseline). It's only enabled while `isDirty`. If the form lives in a dialog or panel, you typically want a Cancel button **outside** `<FormActions>` that closes the panel — that's a parent concern, not a form concern.

## Create form pattern

```tsx
"use client";

import { Input } from "@/components/ui/input";
import {
  FormActions,
  FormField,
  FormProvider,
  useCreateForm,
} from "@/lib/forms";
import { projectsService } from "@/lib/services/projects.service";
import {
  type ProjectCreateInput,
  ProjectCreateInputSchema,
} from "@/lib/types/project";

export function CreateProjectForm({
  onCreated,
}: {
  onCreated: (id: string) => void;
}) {
  const form = useCreateForm<ProjectCreateInput, { id: string }>({
    schema: ProjectCreateInputSchema,
    defaultValues: { name: "", description: "" },
    submit: (value, { signal }) => projectsService.create(value, { signal }),
    onSuccess: (created) => onCreated(created.id),
  });

  return (
    <FormProvider form={form as never}>
      <form
        onSubmit={(e) => {
          e.preventDefault();
          void form.handleSubmit();
        }}
      >
        <FormField name="name" label="Name">
          {(field) => (
            <Input
              id={String(field.name)}
              value={String(field.state.value ?? "")}
              onChange={(e) => field.handleChange(e.target.value)}
              onBlur={field.handleBlur}
              aria-invalid={
                field.state.meta.isTouched && !field.state.meta.isValid
              }
            />
          )}
        </FormField>

        <FormActions submitLabel="Create" submittingLabel="Creating…" />
      </form>
    </FormProvider>
  );
}
```

Create forms gate the submit button on `canSubmit = isValid && !isSubmitting` — dirty tracking is irrelevant because the user is not editing an existing record. (Some teams still gate on `isDirty` to prevent submitting an entirely-default form; that's a per-project preference, codify it once in `useCreateForm` and never per-component.)

## `FormActions`

The shared submit/reset button row. Reads form state reactively via `form.Subscribe` so the rest of the form doesn't re-render when only `isDirty`/`isSubmitting` changes:

```tsx
<FormActions submitLabel="Save" submittingLabel="Saving…" resetLabel="Reset" />
```

If a form needs custom layout for its actions (e.g. a Cancel button between Reset and Save, or actions docked to a dialog footer), drop `<FormActions>` and inline the equivalent:

```tsx
<form.Subscribe selector={(s) => ({
  canSubmit: s.canSubmit,
  isSubmitting: s.isSubmitting,
  isDirty: s.isDirty,
})}>
  {({ canSubmit, isSubmitting, isDirty }) => (
    <div className="flex justify-end gap-2">
      <Button
        type="button"
        variant="ghost"
        disabled={!isDirty || isSubmitting}
        onClick={() => form.reset()}
      >
        Reset
      </Button>
      <Button type="submit" disabled={!canSubmit}>
        {isSubmitting ? "Saving…" : "Save"}
      </Button>
    </div>
  )}
</form.Subscribe>
```

The Subscribe selector is what makes the gating cheap — only this subtree re-renders on state changes.

## Schemas

Schemas live in `lib/types/<entity>.ts` and are the single source of truth for both the form and the service layer. Use Zod v4:

```ts
import * as z from "zod/v4";

export const ProjectSchema = z.object({
  id: z.string().uuid(),
  name: z.string().trim().min(1).max(120),
  description: z.string().trim().max(5000).nullable(),
});
export type Project = z.infer<typeof ProjectSchema>;

export const ProjectCreateInputSchema = ProjectSchema.omit({ id: true });
export type ProjectCreateInput = z.infer<typeof ProjectCreateInputSchema>;
```

Patterns:

- **Edit form** → pass the full entity schema. The hook validates the whole shape on every change; the Save button is disabled until the whole form is valid and dirty.
- **Create form** → pass a dedicated input schema (typically `Schema.omit({ id: true, createdAt: true, … })`).
- **Partial PATCH** → `Schema.partial()` if the backend accepts any subset.

Never write a standalone TypeScript type that mirrors a schema. Always `z.infer<typeof XSchema>`.

## Unsaved-changes guard

Because edit forms only persist on Save, leaving the route or closing the dialog with `isDirty === true` silently discards the user's work. Wire a guard at the boundary:

- **Dialog/panel close**: subscribe to `form.state.isDirty` and prompt before closing. Project-shaped — could be a `confirm()`, a shadcn `<AlertDialog>`, or a "are you sure?" toast.
- **Route navigation**: in Next.js App Router, use a `useRouter`-aware confirm (e.g. `router.events`-equivalent isn't available; the pragmatic answer is `beforeunload` for full-tab close + a custom `<Link>` wrapper for in-app nav). Keep this concern in `lib/forms/` so every form gets it for free.

This skill flags the requirement; the exact UX is project-shaped.

## Error handling

The `save` / `submit` callback **throws**. Do not catch inside it. The hook routes the thrown error through `mapFormError`:

| Error type | UI action |
|---|---|
| `SessionExpiredError` (HTTP 401) | Redirect to your auth refresh route (project-specific). Never toast. |
| `ForbiddenError` (HTTP 403) | Error toast — the user clicked Save expecting it to work; tell them why it didn't. |
| Validation problem (HTTP 422 / RFC 7807 with `errors: { field: [...] }`) | Per-field `errorMap.onServer` set + generic toast. Field errors render under each `<FormField>`. |
| Other server problem (4xx / 5xx) | Toast the `detail` or `title` from the response body. |
| Invalid response shape | Toast a generic "invalid response" message. |
| `TypeError` / network failure | Toast a generic "network error" message. The form stays dirty so the user can retry. |

**Never wrap `save` / `submit` in try/catch.** Never call `toast.success` / `toast.error` from a form component. Never re-implement the discriminated-union routing inline — extend `mapFormError` if you need a new case.

On any error, the hook does **not** reset the baseline — `isDirty` stays `true` so the user can fix and retry without losing their edits.

## i18n

If the project uses `next-intl` (or any other i18n library), all visible strings — labels, placeholders, button text, error messages — go through it. Never inline strings in form code. Add keys as you go. The skill is i18n-agnostic but i18n-mandatory: if a project ships in only one language today, still route through the i18n boundary so adding a second one tomorrow doesn't require rewriting every form.

If the project has no i18n setup at all, that's a separate decision; in that case it's fine to inline strings, but flag it once at the top of the conversation so the user knows the cost they're accepting.

## Red flags / rationalizations

| Rationalization | Counter |
|---|---|
| "I just need a `useState` for one toggle." | Use `<form.Field name="…">` with `<Switch>`. The form hook owns it. |
| "TanStack Form's API is fine here, I'll call `useForm` directly." | No. Always go through `useEditForm` / `useCreateForm` — they wire dirty tracking, baseline reset, error mapping, and AbortController. Bypassing means each form re-derives the same concerns with a slightly different bug. |
| "Auto-save would be nicer, I'll add a 300ms debounce on change." | No. This skill mandates Save buttons with dirty state. Auto-save changes the failure model (silent partial saves, race conditions, no clear user moment of commit) and the UX contract for the whole app. If you genuinely need auto-save, raise it as a project-wide decision; do not introduce it in one component. |
| "I'll save on blur, it feels snappier." | Same problem. Save-on-blur is auto-save with a different trigger. The contract is: explicit Save click. |
| "I'll just use `react-hook-form`, I know it better." | The project standardizes on TanStack Form. Mixing form libraries inside one app is a non-starter. If you genuinely think RHF is a better fit, raise it as a project-wide decision; do not introduce it in one component. |
| "I'll track dirty state with my own `useState` of initial values." | The hook already exposes `isDirty` from `form.state`. Read it via `form.Subscribe`. |
| "I'll re-enable the Save button even when the form is clean — feels weird disabled." | Don't. A clean form has nothing to save; clicking Save would either no-op or POST identical data. Disabled is honest. |
| "I'll try/catch the save and `toast.error` myself." | Never. Errors flow through `mapFormError`. Catching swallows the discriminated-union routing and produces inconsistent UX across forms. |
| "I'll show a success toast on every save." | Configure it once in `mapFormError` (or your hook's success path) so every form behaves identically. Never toast from a form component. |
| "No AbortController needed — the button is disabled while submitting." | The button is the right primary defense, but a stale request can still resolve out-of-order if the user resets and re-submits quickly. Leave the AbortController on. |
| "I need per-field errors, I'll `safeParse` and `setState`." | The hook already runs `validators: { onChange, onBlur }` from your schema. Errors surface via `field.state.meta.errors` and render through `<FormField>`. |
| "I'll catch the 422 and translate the `errors` map myself." | The hook reads the problem-details `errors` map and calls `setFieldMeta` to populate `errorMap.onServer`. Render via `<FormField>` — no extra work needed. |
| "This form is in a Server Component, I'll use Server Actions." | Forms with client-side validation, dirty tracking, and reactive submit gating are inherently client-side. Mark the leaf component `"use client"` and keep the parent page a Server Component. Server Actions for forms are for full-page submits without progressive client UX. |
| "I'll inline copy this once, I'll i18n it later." | The "later" pass touches every form in the app. Add the key now. |

## Escape hatch — `<form.Field>` directly

Drop to raw `<form.Field>` (skipping `<FormField>` wrapping) for:

- File uploads (multipart/form-data, progress, drag-drop UI).
- Multi-step wizard navigation that the form layer can't model linearly.
- Inline table-cell editors (one cell, no surrounding `<FormProvider>` boundary).

Even when escaping `<FormField>`, **still use `useEditForm` / `useCreateForm`** and let `mapFormError` route errors. Still go through your i18n layer. Still gate submit on dirty + valid. The escape hatch is the `<FormField>` wrapper, not the hook contract.

## Toolkit reference

If `lib/forms/` doesn't exist yet in the project, here's the minimum surface to build once. Names match what this skill assumes (`useEditForm`, `useCreateForm`, `FormProvider`, `FormField`, `FormActions`, `mapFormError`).

```
lib/forms/
├── index.ts                  # public re-exports
├── useEditForm.ts            # manual-save hook with dirty tracking + baseline reset
├── useCreateForm.ts          # manual-submit hook with onSuccess
├── FormProvider.tsx          # React context wrapping a TanStack form instance
├── FormField.tsx             # shadcn Field + FieldLabel + FieldError, render-prop API
├── FormActions.tsx           # Save + Reset buttons, gated via form.Subscribe
└── mapFormError.ts           # discriminated-union error → toast / per-field meta
```

Implementation notes — the load-bearing details:

- `useEditForm` wraps `useForm` from `@tanstack/react-form` with `validators: { onChange: schema, onBlur: schema }`. On submit, it awaits the `save` callback (passing a fresh `AbortSignal`), then calls `form.reset(savedValue)` so the new clean state is the saved value. On error, routes through `mapFormError` and **does not** reset — the form stays dirty.
- `useCreateForm` is the same shape minus the baseline reset; on success it calls `onSuccess(result)`.
- Both hooks accept `{ schema, defaultValues, save | submit, onSuccess?, mapError? }` and return the TanStack form instance directly (no wrapper object — `form.state.isDirty`, `form.state.canSubmit`, `form.handleSubmit()` are all on it).
- `<FormField>` is a thin render-prop over `<form.Field name>` — it pulls `useFormContext()` for the form instance, renders `<Field>` + `<FieldLabel>` + `<FieldDescription>` + `<FieldError>`, and exposes the `field` API to the child function.
- `<FormActions>` wraps a single `form.Subscribe` that selects `{ canSubmit, isSubmitting, isDirty }` and renders the Save + Reset buttons. Save is disabled unless `canSubmit && isDirty` (edit) / `canSubmit` (create — pass a `requireDirty={false}` prop or split the component, your call).
- `mapFormError` is a `switch` on the error type. Keep it boring; every form behaves identically because every form goes through it.

The exact file contents are project-shaped (your error classes, your service signatures, your i18n adapter), so the skill doesn't ship a copy — but the surface above is the contract every form in the codebase consumes.

## Audit

When the user asks to **audit** a codebase against this skill (phrasings: "audit my codebase against nextjs-forms", "/audit-forms", "scan for form anti-patterns", "find violations of the form rules"), follow this recipe. Do not modify code during the audit — produce a report only. Offer to fix issues afterwards.

### Step 1 — confirm scope

Run prereq checks first; if any fail, abort the audit and tell the user why:

- `package.json` has `"next": "^16…"` and routes live under `app/`.
- `@tanstack/react-form` is in `dependencies`.
- `react-hook-form` is **not** in `dependencies`. If both are present, that's the first finding — flag it and ask the user which one wins before continuing.
- `components.json` exists at the project root (shadcn/ui configured).
- The required shadcn components are installed (check `components/ui/` for `field.tsx`, `input.tsx`, `textarea.tsx`, `select.tsx`, `checkbox.tsx`, `switch.tsx`, `radio-group.tsx`, `button.tsx`, `label.tsx`, and `sonner.tsx`).
- `lib/forms/index.ts` exists and exports `useEditForm`, `useCreateForm`, `FormProvider`, `FormField`, `FormActions`, `mapFormError`. If the toolkit is missing, that's a top-of-report finding — every other violation downstream is partly caused by its absence.

### Step 2 — scan for the violations

Run these greps in parallel from the project root. Each maps to a rule in this skill. Use ripgrep where available; fall back to `grep -rn`.

```bash
# A. react-hook-form imports anywhere in the codebase
rg -n --type=tsx --type=ts -e "from\s+['\"]react-hook-form['\"]" \
  -g '!node_modules' -g '!.next'

# B. Raw useForm from @tanstack/react-form (bypassing useEditForm/useCreateForm)
rg -n --type=tsx --type=ts -B1 -A2 -e "from\s+['\"]@tanstack/react-form['\"]" \
  -g '!node_modules' -g '!.next' -g '!lib/forms/**'

# C. useState bound to form-like inputs (likely re-implementing field state)
rg -n --type=tsx -B1 -A8 -e 'useState' \
  -g '!node_modules' -g '!.next' \
  | rg -B4 -A4 -e '<Input\b' -e '<Textarea\b' -e '<Select\b' -e '<Checkbox\b' -e '<Switch\b' -e '<RadioGroup\b'

# D. toast.success / toast.error called from form components (look for files that import a form hook AND call toast directly)
rg -ln --type=tsx -e 'useEditForm|useCreateForm' \
  -g '!node_modules' -g '!.next' -g '!lib/forms/**' \
  | xargs -I{} rg -lH --type=tsx -e 'toast\.(success|error)\b' {} 2>/dev/null

# E. Save-on-blur / debounce / auto-save patterns
rg -n --type=tsx --type=ts -B1 -A4 \
  -e 'onBlur.*save' -e 'debounce\(' -e 'setTimeout\(.*save' -e 'autosave|auto-save|autoSave' \
  -g '!node_modules' -g '!.next' -g '!lib/forms/**'

# F. Hand-rolled dirty tracking (initial-values refs, manual diff)
rg -n --type=tsx --type=ts -e 'initialValuesRef' -e 'isDirty\s*=\s*JSON\.stringify' -e 'lastSavedRef' \
  -g '!node_modules' -g '!.next' -g '!lib/forms/**'

# G. <form onSubmit> without going through form.handleSubmit (manual fetch in the submit handler)
rg -n --type=tsx -B1 -A8 -e '<form\s+onSubmit' \
  -g '!node_modules' -g '!.next' \
  | rg -B4 -A4 -e 'fetch\(' -e 'await\s+\w+Service\.' \
  | rg -v 'form\.handleSubmit'

# H. AbortController or signal handling inline in form components (toolkit's job)
rg -n --type=tsx -B1 -A4 -e 'new\s+AbortController\(' \
  $(rg -l --type=tsx -e 'useEditForm|useCreateForm|<form\b' -g '!node_modules' -g '!.next' 2>/dev/null) 2>/dev/null \
  | rg -v 'lib/forms/'

# I. Inline strings on form labels/buttons (i18n bypass) — only if the project uses an i18n library
#    Detect i18n setup first:
rg -ln -e "from\s+['\"]next-intl['\"]" -e "from\s+['\"]react-i18next['\"]" -e "from\s+['\"]@lingui" \
  -g '!node_modules' -g '!.next' >/dev/null && \
rg -n --type=tsx -B1 -A1 -e '<FieldLabel>[^{<]' -e '<Button[^>]*>[A-Z][a-z]' \
  $(rg -l --type=tsx -e 'useEditForm|useCreateForm' -g '!node_modules' -g '!.next' 2>/dev/null) 2>/dev/null

# J. Forms missing dirty-state gating on Save (button always enabled or only gated on isSubmitting)
rg -n --type=tsx -B2 -A6 -e '<Button\s+type=["\x27]submit["\x27]' \
  $(rg -l --type=tsx -e 'useEditForm' -g '!node_modules' -g '!.next' 2>/dev/null) 2>/dev/null \
  | rg -v -e 'isDirty' -e '<FormActions'
```

For each candidate match, open the file and confirm the violation by eye — several patterns above are heuristics that can match legitimate code (e.g. `useState` for non-form UI state in a file that also has a form).

### Step 3 — produce the report

Produce a markdown report grouped by violation kind (A through J above). For each finding, include:

- File path and line number (clickable: `path/to/file.tsx:42`).
- A 3–5 line excerpt showing the offending code.
- Which rule it violates (link to the matching section in this skill).
- The corrected approach in one sentence.
- Severity:
  - **high** — A (mixed form libraries), B (raw `useForm` outside the toolkit), D (inline `toast` from form code), G (manual fetch in submit), J (Save not gated on dirty).
  - **medium** — C (form fields in `useState`), E (auto-save / blur-save / debounce — direct rule violation but sometimes copy-paste legacy), F (hand-rolled dirty tracking), H (inline AbortController).
  - **low** — I (i18n bypass — needs eyes to distinguish UI strings from valid hard-coded values like `aria-label="search"`).

Sort within each group by severity, then file path.

End the report with a summary table:

| Violation | High | Medium | Low | Total |
|---|---|---|---|---|

…and a one-paragraph recommendation. The usual order to fix:

1. If the toolkit is missing (Step 1 finding), build `lib/forms/` first — every other fix depends on it.
2. Pattern A (mixed form libraries) — pick one, removal of the other is a project-wide decision.
3. Pattern B + D (raw `useForm`, inline toasts) — usually the same files; refactor together.
4. Pattern J (missing dirty gating) and E (auto-save) — these change UX, so they're worth a focused PR with a screenshot diff.
5. Patterns C, F, G, H — mechanical refactors once the toolkit is in place.
6. Pattern I — last, after the structural fixes are in.

### Step 4 — offer next steps

After delivering the report, offer (do not execute unprompted):

1. **Build the toolkit** — if `lib/forms/` is missing, scaffold it from the [Toolkit reference](#toolkit-reference) section.
2. **Fix one violation kind at a time** — pick the highest-volume pattern, refactor it across the codebase, run tests/typecheck.
3. **Fix one form at a time** — work top-down through the report, one file per commit.
4. **Open a tracking issue** — convert the report to a GitHub issue with a checklist.

Wait for the user to choose before touching code.

## When in doubt

Re-read this skill, look at the most-recent existing edit form and create form in the codebase, and copy their structure. If the existing forms violate this skill, fix the new one to match the skill (not the existing forms) and flag the drift to the user.
