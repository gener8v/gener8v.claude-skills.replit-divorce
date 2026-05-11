---
name: replit-localize-boot-verification
description: Fix the three cross-platform bugs that Replit-origin projects silently carry — Linux-only `reusePort: true` (crashes on macOS with ENOTSUP), default port 5000 (collides with macOS AirPlay Receiver), and `<meta name="twitter:site" content="@replit" />` in `index.html`. Invoke as part of `replit-localize` Step 5 or directly when the user reports "app crashes on my Mac with ENOTSUP" / "can't bind port 5000". Small, mechanical fixes — usually three single-line edits plus a `.env.example` update.
---

# Boot-Verification Cross-Platform Fixes

Three small bugs that Replit-origin projects ship with because Replit's Linux runtime hides them. None of them surface until a developer tries to actually run the app on macOS for the first time.

## Bug 1 — `reusePort: true` crashes on macOS

Replit's autoscale runtime uses Linux's `SO_REUSEPORT` socket option to load-balance across processes. Replit-Agent often generates `httpServer.listen({ port, host: "0.0.0.0", reusePort: true }, …)`. On macOS:

```
Error: listen ENOTSUP
    at Server.setupListenHandle [as _listen2]
```

Find:

```bash
grep -rE "reusePort" server/ scripts/
```

Fix in `server/index.ts` (or wherever the listen call is):

```ts
// Before
httpServer.listen({ port, host: "0.0.0.0", reusePort: true }, () => { … });

// After
httpServer.listen({ port, host: "0.0.0.0" }, () => { … });
```

The application doesn't need `reusePort` off Replit — Cloud Run, Railway, etc. each run a single process per container and load-balance at the platform level, not the socket level.

## Bug 2 — Port 5000 collides with macOS AirPlay Receiver

macOS Sonoma+ reserves port 5000 for the AirPlay Receiver service (System Settings → General → AirDrop & Handoff → AirPlay Receiver). Replit projects default to port 5000 because Replit's external port mapping is `localPort = 5000 → externalPort = 80`. On macOS:

```
Error: listen EADDRINUSE: address already in use 0.0.0.0:5000
```

Fix: change the local-dev default to a non-5000 port. 5050 is conventional and doesn't collide with anything common.

Two options:

### Option A — Change the code default

In `server/index.ts`:

```ts
const port = parseInt(process.env.PORT || "5050", 10);
```

This means if `PORT` is unset, the dev server starts on 5050.

### Option B — Keep the code default at 5000, override via `.env.local`

In `.env.example`:

```
# macOS: PORT 5000 collides with the AirPlay Receiver service.
# Override to a non-5000 port for local dev.
PORT=5050
```

Option A is friendlier (no env-var override required for fresh-clone setup); Option B keeps the code aligned with Replit's external port mapping. Either works; pick one and document.

**Document the gotcha in README.md too** — every new developer hits this once if they don't know.

## Bug 3 — `@replit` Twitter meta tag in `client/index.html`

Replit-Agent–generated `index.html` files often include:

```html
<meta name="twitter:site" content="@replit" />
```

This is purely cosmetic (no functional impact) but it's a Replit fingerprint that ends up in the production HTML. Remove for a clean divorce.

```bash
grep -rE "@replit" client/index.html public/index.html 2>/dev/null
```

Fix: edit the tag out (or replace with the actual app's Twitter handle if FLS / the product has one).

## Verify

After all three fixes:

```bash
npm run dev
# Server starts cleanly; "serving on port <PORT>" log line appears.

curl -s http://localhost:<PORT>/
# Returns 200 with the SPA shell HTML.

curl -s http://localhost:<PORT>/api/auth/me
# Returns 401 with the expected error JSON (or whatever auth-gate response the app uses).

curl -s http://localhost:<PORT>/ | grep -E "(@replit|twitter:site)"
# Should return nothing — confirms the meta tag was removed.
```

## When to commit

One small commit:

```
fix(local-dev): remove Replit-only socket option and stale meta tag

- server/index.ts: drop reusePort:true (Linux-only, crashes on macOS)
- server/index.ts: default PORT to 5050 (5000 collides with macOS AirPlay)
- client/index.html: remove @replit Twitter meta tag
- README.md: document the port-5000 gotcha for future developers
```

## What this skill doesn't do

- Doesn't address production port configuration. Production reads `PORT` from the platform's injected env (Cloud Run sets `PORT=8080`); the dev default is irrelevant in prod.
- Doesn't audit for other Replit-Agent boilerplate in `client/` — that's a broader cosmetic-cleanup pass not covered here.

## Parent skill

Invoked from `replit-localize` Step 5. Comes after the migration runner is working so the dev server can actually boot to its post-migration state.
