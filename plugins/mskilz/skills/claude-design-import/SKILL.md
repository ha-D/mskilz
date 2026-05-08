---
name: claude-design-import
description: Use when the user gives you a "Claude Design" handoff bundle (downloaded from claude.ai/design) and wants to implement parts of it in the Mirage codebase.
user-invocable: true
---

# Claude Design Import

The user designs visual concepts for Mirage in **Claude Design** (claude.ai/design), an AI design tool that produces HTML/CSS/JS prototypes. They occasionally export a bundle and
ask you to implement parts of it.

This skill captures the workflow so you don't have to re-derive it each time.

## Critical: the codebase is the source of truth

The Mirage codebase has its own established design system, documented in `docs/reference/design-system.md`. The bundle is a **sketch pad** for exploring new ideas — it is **not**
authoritative for foundations.

When the bundle disagrees with the codebase on names, tokens, palette, or conventions:

- **Trust the codebase. Do NOT reconcile back to the bundle's vocabulary.**
- **Do NOT refactor the codebase to match the bundle's foundations.**
- **Do NOT ask the user to adjudicate naming differences** — flag them once in your summary and proceed using the codebase's names.

Example: the bundle uses `--accent` for the brand color; the codebase uses `--brand` (because `--accent` collides with shadcn). When porting a bundle snippet, change
`var(--accent)` → `var(--brand)`. Don't ask. Don't propose renaming `--brand` back. The decision is made.

The user knows the bundle drifts from the codebase over time. They've accepted that. Don't bring it up unless something is genuinely ambiguous about *what to build*.

## What to extract from a bundle, what to ignore

**Extract:**
- New page layouts and view designs
- New component patterns (cards, lists, forms, controls)
- Interaction ideas
- Specific copy text the user spent time iterating on
- Anything in the bundle's chat transcript that shows where the user landed

**Ignore (already codified in the codebase):**
- `colors_and_type.css` foundation file
- The bundle's READMEs covering brand/voice/typography fundamentals
- The "tweaks panel" code (already implemented as `features/appearance/`)
- Any token name choices that conflict with the codebase
- Font choices, color palette, spacing scale (codebase is the source of truth)

If a bundle introduces a *new* token category the codebase doesn't have (e.g. a chart palette, a new severity level), that's worth raising as a question — but lift only the new
one, not the whole foundation.

## Workflow

When the user gives you a bundle:

### 1. Load context first

Before opening the bundle, read:
- `docs/reference/design-system.md` (the codebase's source of truth)
- The relevant feature's working doc in `docs/work/wip/` if applicable

This loads the codebase's vocabulary and conventions so you don't drift toward the bundle's.

### 2. Extract the bundle

Bundles arrive as a tarball at a URL like `https://api.anthropic.com/v1/design/h/...`. Use WebFetch — the response is binary gzip data, which the harness saves to a `.bin` file and reports the path of in its output. Extract that file with `tar -xzf` to a temp directory.

```bash
mkdir -p /tmp/design-extract && cd /tmp/design-extract
tar -xzf <path-to-the-fetched-bin-file>
ls
```

### 3. Read in this order

1. **Top-level `README.md`** — the bundle's "read me first" file (handoff intro for the coding agent)
2. **`chats/chat*.md`** — the conversation transcript. **Don't skip.** Tells you what the user actually wanted, what they explored, what they rejected, and where they landed. The
final HTML files are output; the chat is intent.
3. **The file the user explicitly named to implement** (or `ui_kits/<name>/index.html` if they didn't name one — that's almost always the file they had open when they triggered the
export).
4. **Files that file imports** (CSS, JSX, scripts).
5. **Skim `project/README.md`** for any new ideas worth flagging — but treat its foundations as already-implemented.

### 4. Confirm scope

Bundles are exploratory. The user may not want every page implemented in one pass.

Before writing code, summarize what's in the bundle and ask:
- Which screens/components/patterns from this should land?
- How big should the first pass be?

Use AskUserQuestion if you have several distinct possible scopes. Don't just dive in.

### 5. Implement

When you implement:

- Use the codebase's tokens (`--brand`, `--paper`, `--ink-1`, `--rule`, etc.) — never the bundle's (`--accent`, etc.).
- Use the codebase's existing primitives (shadcn `Button`, `Card`, etc., or established Mirage components).
- If the bundle introduces a new pattern the codebase doesn't have, build it as a new feature module (`apps/web/src/features/<name>/`) following the conventions in AGENTS.md.
- If the bundle's pattern is good but its implementation isn't (raw HTML/CSS prototype), match the visual output, not the structure.

### 6. Document

If you ship a new component pattern or visual convention, add it to the **§5 Component patterns** section of `docs/reference/design-system.md` so the next bundle import doesn't
re-invent it.

## What to do when the bundle's foundations have drifted

The bundle will sometimes have stale foundations (old token names, old preset names, old accent options) because the user iterated in the codebase after exporting it. Examples:

- Bundle says `--accent` for brand → codebase uses `--brand`
- Bundle has `atmospheric` preset → codebase calls it `cream`
- Bundle's tweaks panel offers 13 accents → codebase Settings View already exposes the same 13 via `appearance.accent`
- Bundle uses petrol `#1E5E5A` as the brand → codebase uses forest `#2E5A3A`

**Default action:** silently use the codebase's version. Do not bring it up unless you genuinely cannot resolve what the user wants.

If you must mention a divergence (e.g. the bundle introduces a *new* visual concept that conflicts with an existing codebase decision), flag it once in your summary at the end, in
one short bullet — don't make a section of it.

## Don't try to implement the bundle's `appearance.*` infrastructure

The codebase already has the appearance system (`features/appearance/`, see `design-system.md` §7). If a bundle ships an updated tweaks panel, palette table, or new accent options:

- Update `apps/web/src/features/appearance/tokens.ts` to reflect the new palette data.
- Update `packages/shared/src/settings/definitions.ts` if a setting's enum options changed.
- Do **not** port the bundle's `tweaks-panel.jsx` or `applyTweaks` JS — that's the floating panel approach, which we deliberately rejected in favor of the Settings View
integration.

## Quick checklist before reporting "done"

- [ ] Did I read `docs/reference/design-system.md` before extracting the bundle?
- [ ] Did I read the chat transcript?
- [ ] Did I confirm scope with the user before implementing?
- [ ] Are all colors / fonts / tokens from the codebase, not the bundle?
- [ ] Did I capture any new component patterns in `design-system.md` §5?
- [ ] Did I avoid asking the user to reconcile the bundle's stale foundations?
