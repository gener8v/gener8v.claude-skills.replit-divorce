---
name: replit-audit-security
description: Audit a Replit-origin project for security debt baked in by Replit's UX patterns — plaintext secrets in `.replit` (VLN-001 family) and credentials baked into client bundles by "demo mode" pickers (VLN-002 family). Invoke as part of `replit-audit` or directly when the user asks "any security issues in this Replit project?". Read-only. Produces VLN-N items in the project's vulnerability list.
---

# Replit Security Audit

Two specific audits, each producing one or more `VLN-N` items. These are the two security-debt patterns that show up in nearly every Replit-origin project — not because Replit is insecure, but because its UX patterns nudge developers toward them.

## Audit 1 — Plaintext secrets in `.replit` (VLN-001 family)

Replit's UI encourages users to write secrets into `.replit`'s `[userenv.shared]`, `[userenv.development]`, or `[secrets]` blocks. Anything written there is **plaintext, committed to git, and visible to anyone with repo access** — including anyone who's ever had a fork or contributed to a public repo.

### Recipe

Read `.replit`. List every key-value pair in every env-flavored block. For each, classify:

| Value shape | Verdict |
| --- | --- |
| Empty string | Safe |
| Public URL, region name, feature flag, boolean | Safe |
| API key format (e.g., `sk-…`, `AKIA…`, `xoxb-…`) | **VLN** |
| Anything matching `password`, `secret`, `token`, `key` in the name with a non-trivial value | **VLN** |
| Connection string (`postgres://user:pass@host`) | **VLN** (embedded credentials) |
| Encryption key (typically 32/64 hex chars or base64 of similar length) | **VLN** with extra audit (see below) |

### Extra audit for encryption keys

For any `*_ENCRYPTION_KEY` value, run a two-step audit before recommending rotation:

**Step 1: find the encrypt/decrypt helper callers.**

```bash
grep -rE "(encrypt|decrypt)\\s*\\(" server/ shared/ | head
```

If there are active callers, the key is in use — not just a placeholder.

**Step 2: check column nullability.**

For each encrypted column the helpers write to:

```sql
SELECT column_name, is_nullable FROM information_schema.columns
 WHERE table_schema = 'public' AND column_name LIKE '%encrypted%';
```

(Or grep the schema definition for the column's nullability — `NOT NULL` constraints in the migration or `.notNull()` in a Drizzle schema.)

**Decision tree:**

- No active callers → just a placeholder; rotating is a simple env-var swap.
- Active callers + nullable column → rotation rotates the key going forward; old rows can be left or null-set.
- **Active callers + `NOT NULL` column → rotation requires a re-encryption migration.** You cannot just swap the key; existing rows decrypt with the old key. Document this constraint explicitly in the VLN entry.

### VLN-001 entry format

```markdown
### VLN-NNN: Plaintext `<SECRET_NAME>` in `.replit`

- **Location:** `.replit` line N, block `[userenv.shared]`
- **Value shape:** `<API_KEY|connection_string|encryption_key|generic_secret>`
- **In git history:** Yes (committed in <first-commit-sha>)
- **Severity:** High
- **Active callers:**
  - `server/routes.ts:NNN` (encrypt() helper)
  - `server/routes.ts:NNN` (decrypt() helper)
- **Column nullability** (if encryption key): `clients.ams_data_encrypted` — `NOT NULL`
- **Resolution:** Rotate the value; relocate to target platform's secret manager. **Rotation requires a re-encryption migration** because the column is `NOT NULL` — old rows would fail to decrypt with the new key. Plan a one-shot job that decrypts every row with the old key, re-encrypts with the new key, then commits the new key.
```

## Audit 2 — Bundled credentials in client code (VLN-002 family)

Replit's "demo mode" picker is the canonical offender: a login page that helps developers quickly select a seed user. The convenience pattern is to bake the email-to-password mapping into the client bundle so clicking a user auto-fills both fields. **Anyone who views the page source sees every seed-user password.**

### Recipe

```bash
# Suspicious client-side strings
grep -rE "(password|SeedPass|secret|api[_-]?key)[\"\\s]*[:=]" client/ \
  | grep -v "type =" \
  | grep -v "^\\s*//" \
  | head

# Hardcoded credential maps (object literals with email keys)
grep -rE "['\"][a-z]+@[a-z]+\\.[a-z]+['\"]:\\s*['\"][A-Za-z0-9!@#\\$%]{8,}['\"]" client/

# After build, check the output bundle directly
ls -la dist/public/assets/*.js
grep -E "SeedPass|password.*=.*['\"][A-Z]" dist/public/assets/*.js | head
```

If any of these return hits, you have VLN-002 family items.

### Why the bundle check matters

Even if the client source uses environment variables (`import.meta.env.VITE_SEED_PASSWORD`), Vite **inlines** them at build time. The bundle contains literal strings. Grep the bundle, not just the source.

### VLN-002 entry format

```markdown
### VLN-NNN: Seed credentials bundled into client `<bundle-path>`

- **Location:** `client/login.tsx:NN`, baked into `dist/public/assets/index-<hash>.js`
- **Severity:** High (publicly viewable in any deployed bundle)
- **Affected:** All seed user passwords visible in browser DevTools / view-source.
- **Pattern:** Hardcoded `emailToPassword` map / `SEED_PASSWORD` constant / `import.meta.env.VITE_SEED_PASSWORD` (inlined by Vite at build).
- **Resolution:** Apply the API-driven dev-login picker fix template — see `replit-localize-security-cleanup`. Summary: replace the bundled credential map with a `GET /api/dev/seed-users` endpoint gated by `NODE_ENV !== "production"` that returns only `{ email, role, displayName }` (no credentials); update the picker to `useEffect`-fetch it; the click handler fills *only* the email field; operators type the password manually.
```

## What to write to the audit document

```markdown
## 5. Security debt (VLN-N items)

### VLN-001 family — plaintext secrets in `.replit`
<list each VLN entry as documented above>

### VLN-002 family — bundled credentials in client
<list each VLN entry>

### Summary
- N high-severity items
- N medium-severity items
- N items requiring re-encryption-migration before rotation
```

## What this skill doesn't do

- Doesn't rotate or change any credentials. Rotation happens in `replit-localize-security-cleanup` for non-encryption-key secrets, and as a separately scoped re-encryption migration for encryption keys with NOT NULL columns.
- Doesn't audit non-Replit-specific security issues (SQL injection, missing CSRF, etc.). Run a general security review for those — see Claude Code's `security-review` skill if installed.

## Parent skill

Invoked from `replit-audit`. Feeds the security-cleanup work in Phase 2 (`replit-localize-security-cleanup`).
