---
name: replit-localize-dep-cleanup
description: Right-size a Replit-origin project's `package.json` — remove truly-unused dependencies (Replit-Agent template ghosts, dead build-allowlist entries) and reclassify client-only React/UI deps from `dependencies` to `devDependencies` so they're not shipped in the runtime container. Invoke as part of `replit-localize` Step 7 or directly when the user reports "Docker image is 1 GB" / "production node_modules is huge." Modifies `package.json` and `package-lock.json`.
---

# Dependency Cleanup — Truly-Unused + Client-Only

Replit-Agent fullstack templates universally do two things that make production containers obese:

1. **List packages in `dependencies` that the code never imports.** Template ghosts: `passport`, `passport-local`, `connect-pg-simple`, `memorystore`, `express-session` (when sessions are actually custom); `@google/generative-ai`, `axios`, `cors`, `stripe`, `uuid`, `xlsx` (when none of these are imported); etc. The list varies but the pattern is consistent.

2. **Put client-only React/UI packages in `dependencies` instead of `devDependencies`.** `react`, `react-dom`, all 28+ `@radix-ui/*`, `framer-motion`, `recharts`, `@tanstack/react-query`, `cmdk`, `vaul`, `sonner`, `lucide-react`, `wouter`, `zustand`. Vite bundles all of these into `dist/public/` at build time — the runtime Node container never imports them. But `npm ci --omit=dev` faithfully installs every entry in `dependencies`, so the runtime image ships ~700 MB of dead weight.

This skill addresses both. Run after `replit-audit-deps` has produced the list of Replit-coupled deps.

## Pass 1 — Truly-unused deps

For each package in `dependencies`, check if any source file imports it.

```bash
# Standard import grep — catches `from "<pkg>"` and `require("<pkg>")`
for pkg in $(jq -r '.dependencies | keys[]' package.json); do
  hits=$(grep -rE "(from|require)\\s*\\(?['\"]${pkg}['\"]" \
    server/ scripts/ shared/ client/ 2>/dev/null | grep -v node_modules | wc -l)
  [ "$hits" = "0" ] && echo "UNUSED: $pkg"
done
```

For any flagged package, check dynamic imports before declaring it unused:

```bash
# Catch await import("pkg") and similar
grep -rE "['\"]<pkg>['\"/]" server/ scripts/ shared/ 2>/dev/null
```

Real example: `archiver` showed zero hits in the standard grep but is used via `const archiver = (await import("archiver")).default` in routes. The broader pattern catches it. **Apply investigate-before-changing rigorously** — every removal must be backed by both greps showing zero non-trivial hits.

Also check the **build allowlist** in `script/build.ts` (or wherever esbuild's `external` list is configured):

```bash
grep -A 20 "allowlist\|external" script/build.ts
```

Packages in the allowlist but with zero import sites are "ghost allowlist entries" — Replit Agent left them as template defaults. Common offenders: `date-fns`, `ws`, `zod-validation-error`, `@jridgewell/trace-mapping`. Remove from both `package.json` deps AND the build allowlist (they should match).

For `ws` specifically: also check `optionalDependencies.bufferutil` — it's a transitive perf optimization for `ws`. If `ws` goes, `bufferutil` goes too.

## Pass 2 — Client-only deps misclassified as `dependencies`

The deterministic recipe — every package imported by non-client code:

```bash
grep -rhoE "from ['\"]([@a-z0-9][^'\"./][^'\"]*)['\"]" \
    server/ scripts/ shared/ script/ 2>/dev/null \
  | sed -E "s/from ['\"]//; s/['\"]\$//" \
  | sed -E 's|^(@[^/]+/[^/]+).*|\1|; s|^([^@/][^/]*).*|\1|' \
  | sort -u
```

This produces a clean list of ~10–15 packages — the true runtime surface. Anything in `dependencies` but NOT in this list belongs in `devDependencies`:

- Vite still picks them up at build time and bundles them into `dist/public/`.
- `npm ci --omit=dev` no longer installs them into the production container.
- Bundle sizes are unchanged (proof that the move was correct).

**Borderline cases that need extra checks:**

- **Dynamic imports** — same broader grep as Pass 1. `archiver` looks unused under the standard pattern but is used dynamically.
- **Shared schema imports of zod helpers** — `drizzle-zod` is imported by `shared/schema.ts` and surfaces in the grep. Keep it in `dependencies`.
- **Build-tool support packages** — `esbuild`, `vite` themselves stay in `devDependencies` (they're the build tools, not runtime deps).

## Pass 3 — `@types/*` packages

`@types/*` packages are TypeScript-only and **never imported at runtime** — they always belong in `devDependencies`, regardless of where the underlying JS package lives. Replit-Agent templates routinely list them in `dependencies` next to the main package. Move them all in the same pass.

```bash
jq -r '.dependencies | keys[]' package.json | grep '^@types/'
```

Every one of those moves to `devDependencies`.

## Apply, verify, commit

After all three passes, write the new `package.json`. Run:

```bash
npm install        # updates package-lock.json
npm run check      # TypeScript still passes
npm run build      # builds successfully
```

Compare build outputs to before the cleanup:

```bash
ls -la dist/index.cjs dist/public/assets/*.js
```

**Bundle sizes should be identical to pre-cleanup.** If the server bundle shrinks unexpectedly, you removed a package the server actually used (broader grep missed it). If the client bundle changes, Vite wasn't picking up something — investigate.

Then measure the production-only install:

```bash
mkdir /tmp/prod-install && cd /tmp/prod-install
cp /path/to/project/package.json /path/to/project/package-lock.json .
npm ci --omit=dev --no-audit --no-fund
du -sh node_modules
```

Typical Replit-Agent template before cleanup: ~700 MB. After: 80–120 MB.

## Commit

One commit covers all three passes:

```
chore(deps): right-size dependencies for runtime container

- Pass 1: removed N truly-unused deps (foo, bar, baz, …). Each verified via
  standard import grep AND dynamic-import grep showing zero hits. Also
  pruned the matching entries from script/build.ts's esbuild allowlist.
- Pass 2: moved N client-only React/UI deps from dependencies to
  devDependencies (@radix-ui/*, react, react-dom, framer-motion, …). Vite
  still bundles them; `npm ci --omit=dev` no longer installs them.
- Pass 3: moved all @types/* packages to devDependencies.

Verified: npm run check passes; npm run build produces identical dist/
output sizes (server 1.6 MB, client 1.2 MB JS). Production-only install
shrank from ~700 MB to ~90 MB node_modules.
```

## What this skill doesn't do

- Doesn't address the production Docker image base or build itself. That's `replit-to-gcp-dockerfile` — but this cleanup is a prerequisite for an efficient image.
- Doesn't audit for security vulnerabilities in remaining dependencies. Run `npm audit` separately.
- Doesn't decide whether a dep with a few real callers should be kept vs replaced with stdlib. That's a separate refactoring conversation.

## Parent skill

Invoked from `replit-localize` Step 7. The pre-requisite for `replit-to-gcp-dockerfile` — without this cleanup, the runtime image is ~1 GB. With it, ~450 MB.
