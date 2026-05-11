---
name: replit-audit-config
description: Audit Replit-specific configuration files — `.replit`, `replit.md`, `replit.nix` — for deployment coupling, hook misconfiguration, and plaintext secrets. Invoke as the first audit pass against a Replit-origin project, or directly when a user asks "what's in this .replit file?" / "are there secrets in the Replit config?". Read-only. Outputs findings to the project's audit document.
---

# Replit Configuration File Audit

Audits the three Replit-specific configuration files and produces findings for the project's audit document.

## What to read

Read each file in full, even if it's long.

### `.replit` — primary platform configuration

The most important file. Look for these blocks:

| Block | What it tells you | What to flag |
| --- | --- | --- |
| `[deployment]` | The canonical build/run command Replit uses | Any other build scripts in the repo NOT referenced here are orphaned (e.g., `script/build-prod.sh`). |
| `[userenv.shared]`, `[userenv.development]`, `[secrets]` | Environment variables Replit's UI surfaces. **Universal source of plaintext secrets.** | Inspect every value. Anything that looks like a credential (key, password, token, secret) → flag as `VLN` item. |
| `[postMerge]`, `[onBoot]`, `[onShutdown]` hooks | Replit IDE features that auto-run commands | Often contradict documented "never push" rules in `replit.md`. Document the contradiction explicitly. |
| `[[ports]]` mappings | Port forwarding rules | Usually `localPort = 5000 → externalPort = 80`. Replit projects defaulting to port 5000 collide with macOS AirPlay Receiver — flag for the `replit-localize` boot-verification pass. |
| `[agent]` block | Replit Agent template fingerprint | `stack = "MOCKUP_JS"` + `mockupState = "FULLSTACK"` = canonical Replit-Agent fullstack project. Record this for the audit's project profile. |

### `replit.md` — Replit-Agent-generated project description

Treat as partially-stale background documentation, not authoritative. Cross-check against:

- Actual code state (does what `replit.md` says match what the code does?)
- Recent commits (has the code drifted from the description?)
- Other docs in `docs/` (are they more current?)

Common drift patterns: deployment commands have changed since `replit.md` was written, "never run X" rules contradict `[postMerge]` hooks in `.replit`, the architecture diagram references services that no longer exist.

Recommendation: after the audit, rename or replace `replit.md` with a non-Replit-named doc (`docs/PROJECT_CONTEXT.md` is the canonical replacement) — this is done in `replit-localize`, not here.

### `replit.nix` if present

System packages Replit pre-installs. Almost always portable — the same packages can be installed via `apt-get`, `apk`, or a Dockerfile. Note any unusual ones (custom binaries, specific Python versions) — these may need Dockerfile entries during `replit-to-gcp`.

## What to write to the audit document

Add a section to `docs/REPLIT_AUDIT.md` (or whatever the audit doc is named):

```markdown
## 1. Replit configuration files

### `.replit`
- **Deployment target:** <e.g., autoscale> (canonical build/run: `<command from [deployment]>`)
- **Hooks present:** <list>
- **Plaintext secrets in `[userenv*]` / `[secrets]`:** <count, with VLN-N refs>
- **Port mapping:** <localPort → externalPort>
- **Agent fingerprint:** <stack/mockupState if present>

### `replit.md`
- **State:** authoritative / partially-stale / stale
- **Drift items:** <list of statements that don't match current code>

### `replit.nix`
- **System packages:** <list>
- **Non-portable items:** <none / list>
```

## VLN items produced by this pass

If plaintext secrets are found in `.replit`, create or update the project's vulnerability list with one entry per secret:

```markdown
### VLN-NNN: Plaintext <SECRET_NAME> in `.replit`

- **Location:** `.replit` line N, block `[userenv.shared]`
- **Severity:** High (committed to git history)
- **Resolution:** Rotate the value; relocate the production version to the target platform's secret manager. If the value is an encryption key, see `replit-audit-security` for the re-encryption-migration consideration.
```

## What this skill doesn't do

- Doesn't modify any files. The `.replit`, `replit.md`, and `replit.nix` files stay as-is until `replit-localize` rebuilds them.
- Doesn't audit Replit-related code paths (`@replit/*` packages, env-var usage) — those are separate sub-skills (`replit-audit-deps`, `replit-audit-env-vars`).
- Doesn't decide whether to migrate. That's a strategic decision; this audit informs it.

## Parent skill

This sub-skill is invoked from `replit-audit` (Phase 1 of the Replit-divorce workflow). The parent skill sequences the six audit passes and produces the consolidated inventory document.
