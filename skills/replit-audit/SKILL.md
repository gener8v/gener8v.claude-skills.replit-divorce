---
name: replit-audit
description: Phase 1 of the Replit-divorce workflow. Coordinates the five sub-skills that produce a complete portability and security inventory of a Replit-origin project. Use when the user asks to "audit this Replit project," "inventory what's tying us to Replit," or before any Replit→cloud migration. Read-only — modifications happen in Phase 2 (`replit-localize`). Output is a written audit document.
---

# Replit Project Audit — Phase 1 Coordinator

You are running Phase 1 of the Replit-divorce workflow: a discovery pass that produces a written inventory the user can act on. **This skill does not modify files.** Modifications happen in Phase 2 (`replit-localize`).

The work is split across **five sub-skills**. Invoke them in the order below, each producing a section of the final audit document.

## Fingerprint check (do this first)

Before running the sub-skills, confirm you're looking at a Replit-Agent-fullstack-shaped project. The canonical fingerprint:

- `[agent].stack = "MOCKUP_JS"` and `mockupState = "FULLSTACK"` in `.replit`
- Node + Vite + Postgres + an `autoscale` deployment target
- Three `@replit/vite-plugin-*` dev-deps (`cartographer`, `dev-banner`, `runtime-error-modal`)
- `replit.md` at repo root with Replit-Agent-flavored project description

If three or more are present, the sub-skills below apply almost verbatim. If fewer, the project is still Replit-origin but may have non-standard shape — proceed with extra attention to false-negative greps.

## Sub-skill sequence

Run in this order:

| # | Sub-skill | Output section |
| --- | --- | --- |
| 1 | [`replit-audit-config`](../replit-audit-config/SKILL.md) | §1 Replit configuration files |
| 2 | [`replit-audit-deps`](../replit-audit-deps/SKILL.md) | §2 Replit-specific dependencies |
| 3 | [`replit-audit-env-vars`](../replit-audit-env-vars/SKILL.md) | §3 Environment variable contract |
| 4 | [`replit-audit-lifecycle-scripts`](../replit-audit-lifecycle-scripts/SKILL.md) | §4 Lifecycle scripts |
| 5 | [`replit-audit-security`](../replit-audit-security/SKILL.md) | §5 Security debt (VLN-N items) |

Each sub-skill is independently invokable — if the user only wants the env-var contract, invoke just `replit-audit-env-vars`. But for a full Phase 1 audit, run all five.

## Output document

All five sub-skills append to the same document — recommend `docs/REPLIT_AUDIT.md` (or `docs/INFRASTRUCTURE.md`, which is the canonical case-study name). Structure:

```markdown
# Replit Audit — <Project Name>

**Date:** <YYYY-MM-DD>
**Fingerprint:** <which Replit-Agent template signals are present>

## 1. Replit configuration files
<from replit-audit-config>

## 2. Replit-specific dependencies
<from replit-audit-deps>

## 3. Environment variable contract
<from replit-audit-env-vars>

## 4. Lifecycle scripts
<from replit-audit-lifecycle-scripts>

## 5. Security debt (VLN-N items)
<from replit-audit-security>

## 6. Recommendations summary
<consolidated, severity-ranked recommendations>
```

After running all five sub-skills, write §6 yourself — a consolidated severity-ranked recommendation list pointing each item at the Phase 2 sub-skill that fixes it.

## Severity scale for the inventory

Use this scale consistently across all five sub-skills' outputs:

| Severity | Meaning |
| --- | --- |
| **🔴 Hard-blocking** | Breaks the build/runtime off Replit (e.g., `*-gate.sh` scripts that grep `.replit`; missing required env vars without fallbacks). |
| **🟡 Behavioral** | Silently produces wrong output off Replit (e.g., un-gated `runtime-error-modal` plugin; `DATABASE_URL`-fallback HMAC seed; `reusePort: true` on macOS). |
| **🟢 Cosmetic** | Doesn't affect function but signals Replit origin (e.g., `@replit` Twitter meta tag, "Created by Replit Agent" comments). |

`VLN-N` items are tracked separately from the severity scale — they have their own High/Medium/Low classification per standard CVE-style vulnerability scoring.

## The meta-discipline: investigate-before-changing

This applies to every audit pass and is the most important rule in the whole skill pack:

> When something *looks* like cruft or unused dependency, **grep for actual usage before mutating it.**

The audit's job is to *recommend* changes, not to make them; recommendations must be grounded in actual code state, not appearance. Two real cases:

- `@anthropic-ai/claude-code` *looked* unused → grep confirmed zero imports → safe to recommend removal.
- `AMS_SYNC_ENCRYPTION_KEY` *looked* like a placeholder string → grep showed active callers and a `NOT NULL` column → rotation requires a re-encryption migration, not an env-var swap.

Every "remove this" recommendation in the audit document must be accompanied by the grep evidence that proves it's safe.

## When you're done

Return a short summary to the user covering:

1. The fingerprint (one line — "this is a Replit-Agent fullstack project with the canonical shape").
2. Count of items by severity (🔴 / 🟡 / 🟢) and `VLN-N` count.
3. Top 3–5 highest-severity findings.
4. Pointer to the full audit document.
5. Recommend invoking `replit-localize` (Phase 2) next, or addressing specific findings first.

## Parent skill

This is invoked from `replit-divorce` (the top-level orchestrator). On completion, return control to the orchestrator for the decision on whether to proceed to `replit-localize`.
