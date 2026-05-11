---
name: replit-localize-security-cleanup
description: Resolve VLN-001 (plaintext secrets in `.replit`) and VLN-002 (credentials bundled into client code) findings from `replit-audit-security`. Invoke as part of `replit-localize` Step 6 or directly when the user asks "fix the credential leaks in this Replit project." Modifies code and rotates secrets. Does NOT apply to encryption keys with NOT NULL columns (those need a re-encryption migration handled separately).
---

# Security Cleanup — VLN-001 and VLN-002

Applies the standard fix templates for the two security-debt patterns the audit produces.

## VLN-001 — Plaintext secrets in `.replit`

The audit (`replit-audit-security`) produced a list of secret-shaped values found in `.replit`'s `[userenv.*]` / `[secrets]` blocks.

For each entry on the list, follow the rotation playbook:

### Standard rotation (most secrets)

1. **Generate a fresh value.** For session secrets, signing keys, API tokens — produce a new random value of appropriate length. For external service credentials (R2, Anthropic, Stripe), regenerate from the service's dashboard and revoke the old.
2. **Stage the new value in the target platform's secret manager.** Don't touch the running production until the new value is in place. This is platform-specific work — Cloud Run uses Secret Manager (see `replit-to-gcp-terraform-secrets`); Railway uses its panel; etc.
3. **Cut over.** Update the running service to use the new value. Verify.
4. **Revoke the old.** Now safe to deactivate at the source.
5. **Remove from `.replit`** — and rewrite git history if the leakage window is significant. Note that rewriting history is destructive; for a private repo with limited contributors, `BFG Repo-Cleaner` or `git filter-repo` is the standard tool. For public history, treat the secret as compromised and rotate, period.

### Encryption-key rotation — STOP and defer

If the audit flagged any `*_ENCRYPTION_KEY` with active callers AND a NOT NULL encrypted column, **rotation requires a re-encryption migration.** Do NOT just swap the env var — every encrypted row in the DB was written with the old key; decryption with the new key will produce garbage.

The re-encryption migration is a separate, planned piece of work:

1. Write a one-shot job that connects to the DB with the old key in env.
2. For each encrypted row: decrypt with old key, re-encrypt with new key, UPDATE.
3. Run the job once, transactionally per table.
4. Then swap the env var.

This work is **out of scope for `replit-localize`**. Document the constraint in `OPEN_QUESTIONS.md` and surface to the user as a separate piece of follow-on work. Treat the old key as still-in-use until the re-encryption migration runs.

## VLN-002 — Bundled credentials in client code

The audit's VLN-002 entries identified a "demo mode" login picker that bakes seed-user credentials into the client bundle. Fix template — apply once and it covers the canonical pattern:

### Step 1 — Add the API endpoint

In your routes file (e.g., `server/routes.ts`), register a route gated by environment:

```ts
if (process.env.NODE_ENV !== "production") {
  app.get("/api/dev/seed-users", async (_req, res) => {
    const users = await db
      .select({
        email: usersTable.email,
        role: usersTable.role,
        displayName: usersTable.displayName,
      })
      .from(usersTable)
      .where(/* filter to seed users only, e.g., domain match */);

    res.json({ users });
  });
}
```

Critical:

- **Gate with `NODE_ENV !== "production"`.** esbuild bakes `NODE_ENV` into the production bundle at build time; the route literally won't exist in production builds.
- **Return only `{ email, role, displayName }`** — no password, no token, no anything that could authenticate.
- **Filter to seed users only** — don't expose real customer email addresses. Match on a known seed-user pattern (e.g., `@ask-test.dev` domain) or a `is_seed` flag.

### Step 2 — Replace the bundled credentials map in the client

Find the existing login picker component. It currently imports a hardcoded map (a `Record<string, string>` or similar). Replace with:

```tsx
const [seedUsers, setSeedUsers] = useState<SeedUser[]>([]);

useEffect(() => {
  if (import.meta.env.VITE_DEMO_MODE !== "true") return;
  fetch("/api/dev/seed-users")
    .then((r) => r.json())
    .then(({ users }) => setSeedUsers(users))
    .catch(() => setSeedUsers([]));   // fail silently — picker just won't show
}, []);

// Picker UI: render seedUsers as clickable rows.
// Click handler: ONLY fill the email field. NEVER fill the password.

function handleSeedUserClick(user: SeedUser) {
  setLoginForm({ ...loginForm, email: user.email });
  // password stays whatever the user typed (or empty)
}
```

The click handler fills **only the email field.** The operator types the password manually — which they know from the seed-script source or out-of-band documentation in `docs/PROJECT_CONTEXT.md`.

### Step 3 — Remove the bundled credentials file

Delete the file (`client/seed-users.ts` or wherever the map lived). Confirm no other imports reference it:

```bash
grep -rE "seed-users|SEED_PASSWORD" client/ shared/
```

Zero hits expected.

### Step 4 — Build and verify

```bash
npm run build
grep -E "(SeedPass|password.*['\"][A-Z])" dist/public/assets/*.js | head
```

Zero matches expected. If anything turns up, find the source and remove.

Then run the dev server and confirm:

1. With `VITE_DEMO_MODE=true` (local dev): the picker appears, click fills only email, you type the seed password manually, login works.
2. With `VITE_DEMO_MODE=false` or unset (production-like): picker doesn't appear.
3. Hit `/api/dev/seed-users` directly with `NODE_ENV=production` build: route returns 404 (it's not registered).

### Step 5 — Document the seed password contract

In `docs/PROJECT_CONTEXT.md` (or equivalent project doc — NOT in any bundle), document the seed-user password for local dev. Operators read it from documentation, not from the bundle.

```markdown
## Local dev login

Seed users (created by `npm run db:seed`):

| Email | Role |
| --- | --- |
| alex.platform@ask-test.dev | platform_admin |
| laura.leader@ask-test.dev | agency_leader |
| … |

All seed users share the password `<DocumentedSeedPassword>`. The picker
fills only the email; type this password manually.
```

## When to commit

After both VLN classes are addressed:

```
fix(security): drive dev login picker from API, not bundle (VLN-002)

- Add GET /api/dev/seed-users gated by NODE_ENV !== "production"
- Replace bundled credentials map in client login picker with useEffect fetch
- Click handler fills email only; operator types password manually
- Delete client/seed-users.ts and confirm grep finds no residue in dist/
- Document seed password contract in docs/PROJECT_CONTEXT.md

Resolves: VLN-002
```

For VLN-001 — typically a separate commit per rotated secret if the rotation involves git-history rewriting; otherwise bundled into the platform-side secret-population work (`replit-to-gcp-terraform-secrets`).

## What this skill doesn't do

- Doesn't perform the re-encryption migration for `*_ENCRYPTION_KEY` rotations with NOT NULL columns. Surface as `OPEN_QUESTIONS.md` item; coordinate as separate work.
- Doesn't audit for non-Replit-Agent security patterns (SQL injection, missing CSRF tokens, etc.). General security review is a separate concern.

## Parent skill

Invoked from `replit-localize` Step 6. The audit's VLN-N list (from `replit-audit-security`) is the input; resolved entries get marked accordingly in the project's vulnerability list.
