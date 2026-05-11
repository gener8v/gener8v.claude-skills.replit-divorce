---
name: replit-audit-deps
description: Audit a Replit-origin project's npm/yarn dependencies for Replit-specific packages, Replit-Agent residue (`@anthropic-ai/claude-code`), and project-local plugins that read Replit env vars. Invoke as part of `replit-audit` or directly when the user asks "what Replit packages does this use?" / "are there Replit-specific deps?". Read-only.
---

# Replit Dependency Audit

Identifies every npm package and project-local plugin that ties the codebase to Replit's environment.

## Three categories to find

### 1. `@replit/*` packages

The three Replit Vite plugins almost always co-occur in Replit-Agent fullstack projects:

```bash
grep -rE "@replit/vite-plugin-(cartographer|dev-banner|runtime-error-modal)" .
```

Each has different mitigation behavior:

| Plugin | Typical gating | Recommendation |
| --- | --- | --- |
| `@replit/vite-plugin-cartographer` | `REPL_ID`-gated import | Safe to leave (no-op off-Replit). Document the gate. |
| `@replit/vite-plugin-dev-banner` | `REPL_ID`-gated import | Same — no-op off-Replit. |
| `@replit/vite-plugin-runtime-error-modal` | Often **NOT** gated | Audit the import site. Recommend `REPL_ID`-gating, or remove entirely. |

Also check for any other `@replit/*` package not in this canonical three — `package.json` and `package-lock.json` are the authoritative sources.

### 2. Replit-Agent residue

Replit Agent–created projects often have `@anthropic-ai/claude-code` listed in runtime `dependencies`:

```bash
grep -E '"@anthropic-ai/claude-code"' package.json
```

This package shouldn't be in a production container. Verify it's actually unused with the standard import grep:

```bash
grep -rE "(from|require)\\s*\\(?['\"]@anthropic-ai/claude-code" server/ scripts/ shared/ client/ 2>/dev/null
```

Zero hits = safe to flag for removal. Apply the **investigate-before-changing** discipline — don't recommend removal without confirming via grep.

### 3. Project-local plugins that read Replit env vars

These are the easiest to miss. They hide as innocent-looking Vite plugins or build scripts but reference Replit-specific environment variables.

```bash
grep -rE "REPLIT_(INTERNAL_APP_DOMAIN|DEV_DOMAIN|DB_URL|HOME|SLUG|OWNER)" .
```

Common offender pattern: `vite-plugin-meta-images.ts` or similar plugin that generates og:image meta tags using Replit's domain env vars. Replit's own "deploy with og:image" cookbook recipe produces this exact pattern. Auditing only `@replit/*` misses it.

For each finding, document:
- File path
- Which Replit env var is referenced
- What the plugin/script does with it
- Whether off-Replit behavior is graceful (falls back to a non-Replit value) or broken (errors / produces wrong output)

## What to write to the audit document

```markdown
## 2. Replit-specific dependencies

### npm packages
| Package | Version | Type | Gating | Recommendation |
| --- | --- | --- | --- | --- |
| `@replit/vite-plugin-cartographer` | … | devDep | REPL_ID-gated | Leave — no-op off Replit |
| `@replit/vite-plugin-runtime-error-modal` | … | devDep | **NOT GATED** | Add REPL_ID gate or remove |
| `@anthropic-ai/claude-code` | … | dep | n/a | Remove (unused; Replit Agent residue) |

### Project-local Replit-coupled plugins
| File | Replit env var | Off-Replit behavior |
| --- | --- | --- |
| `vite-plugin-meta-images.ts` | `REPLIT_INTERNAL_APP_DOMAIN`, `REPLIT_DEV_DOMAIN` | Falls back silently; meta-images don't render |
```

## VLN items produced by this pass

None typically. Replit packages and Replit-Agent residue are coupling issues, not vulnerabilities. The exception: if `@anthropic-ai/claude-code` ships in a production container with a hardcoded API key or other credential material, that's a `VLN` item.

## What this skill doesn't do

- Doesn't modify `package.json` or remove anything. Removal happens in `replit-localize-dep-cleanup`.
- Doesn't audit non-Replit dependencies for general dead code. The broader "what's unused" pass is also in `replit-localize-dep-cleanup`.

## Parent skill

Invoked from `replit-audit`. The output feeds the dep-cleanup work in Phase 2 (`replit-localize-dep-cleanup`).
