# Claude Invariants — Internal Technical Reference

> **🤖 This file is for Claude only.** It documents technical rules I (Claude) must follow when writing code for this skill. The user does not need to read this; the user-facing experience is "analytics works" or "it doesn't." These invariants exist because each one represents a silent-failure mode where production breaks without visible errors.

## When to read this file

I read this file:
- Once at the start of any session where I'm executing this skill end-to-end
- Whenever a stage file references an invariant by number ("see Invariant #3 in `_internal/`")
- When troubleshooting a production failure that resembles one of these silent-failure modes

The user never sees this file unless they explicitly ask "what are the technical invariants?"

## The six invariants

### 1. Vite bakes `VITE_*` env vars at BUILD TIME — not runtime

If they're missing when Railway runs `npm run build`, the deployed bundle is silently dark. Setting them after deploy and reloading does NOT fix it — must trigger a rebuild.

```js
// What I wrote:
const key = import.meta.env.VITE_POSTHOG_KEY
posthog.init(key, { ... })

// What ships in the bundle if VITE_POSTHOG_KEY was unset at build:
const key = undefined
posthog.init(undefined, { ... })  // silently fails — no error, no warning
```

**My check**: I view the deployed bundle source via Bash and grep for the literal token (e.g., `phc_xxxxx`). If absent, the env var wasn't in the build environment. I trigger a rebuild with vars set.

**Setting env vars on Railway** (verified 2026-05 against `docs.railway.com/ai/mcp-server` and `docs.railway.com/cli`):

- **MCP tools that exist**: `set-variables`, `list-variables` — these work for non-sensitive values (port numbers, URLs, public keys).
- **CLI form**: `railway variable set KEY=value` (singular `variable`, not `variables`) and `railway variable list` (or `railway variable list --json` for structured output). After a one-time `railway login` browser OAuth, every subsequent CLI call is autonomous.
- **My default for SECRETS** (Resend keys, Whop secrets, etc.): CLI with shell substitution — `railway variable set RESEND_API_KEY="$(cat .secrets/resend-api-key.txt)"`. The substitution happens before `railway` sees the args, keeping the secret out of my chat history. The MCP `set-variables` path would require me to read the secret file's contents into my context first, then send via MCP — which means the secret flows through Claude's context window. That's worse for leak defense, so for any value I treat as sensitive, the CLI route wins.
- **My default for non-secrets** (`PORT`, `SITE_URL`, `APP_URL`, `RAILWAY_CHECKOUT_DEPTH`, etc.): either path is fine; I lean on the CLI for consistency with the secrets path so all env-var ops in a stage use the same surface.

Full pattern + secrets-file handling in [stage-4-railway.md](../stage-4-railway.md) Step 3.

### 2. PostHog reverse proxy MUST be mounted BEFORE `express.json()`

PostHog sends compressed binary; the proxy needs raw bytes. Same applies to webhook handlers needing HMAC verification. Once `express.json()` parses the body, the bytes are gone.

```js
// CORRECT order — I verify this in every server.js I write:
registerPosthogProxy(app, { staticDir })   // raw body needed
registerWhopWebhook(app)                    // raw body needed for HMAC
app.use(express.json({ limit: "32kb" }))   // now safe to parse JSON
app.post("/api/newsletter", ...)            // body parser already ran
```

### 3. Super-properties register BEFORE first capture

`posthog.register()` only attaches properties to events fired *after* it's called. A common bug: registering deploy/variant tags in a `useEffect` that fires after `posthog.init()`'s initial `$pageview`, leaving bounce visitors' single event tagged with `deploy_variant: null`.

The right place is INSIDE `posthog.init()`'s `loaded` callback, BEFORE the first `ph.capture('$pageview', ...)`:

```js
posthog.init(key, {
  api_host: host,
  capture_pageview: false,        // I'll fire it manually
  loaded: (ph) => {
    registerDeployTagOnPostHog()  // ← BEFORE capture
    registerCustomUtmsOnPostHog()
    registerAffiliateCodeOnPostHog()
    ph.capture('$pageview', { ... })  // ← now super-properties attach
    initAnalytics()  // Clarity + flag-bridge + Clarity-side tags
  },
})
```

### 4. `/static/*` is served LOCALLY, not proxied

`us-assets.i.posthog.com` triggers Cloudflare Error 1000 on Railway → PostHog egress (CF anti-loop heuristic when both sides are CF customers). I vendor `posthog-js/dist/*.js` at build time via a `predev`/`prebuild` script and serve from `express.static`.

### 5. Three Clarity masking layers are belt-and-suspenders

`.clarity-mask` CSS class + `data-clarity-mask="True"` attribute + dashboard `Balanced` masking mode. If any single tool changes its convention, the other two still fire. Marketing sites without user-authored content can use the simpler env-only gate; apps with user content need consent + route-scope gates too.

### 6. Privacy posture is a project-start decision

Switching the consent posture mid-flight means re-prompting all users (consent version bump) and may invalidate previously-collected data depending on the change direction. I lock the posture in **Stage 8** before going live.

## Other technical rules I follow

- **Cross-platform shell**: I use my Bash tool which abstracts Windows/macOS/Linux. I avoid `$()` substitution patterns when documenting commands (PowerShell incompatibility).
- **Placeholder substitution**: when stage files show `<your-domain>` / `<your-proxy-slug>` / `<brand-prefix>` etc., these are placeholders I substitute with the user's actual values at write time (from saved skill config). They should NEVER appear in shipped code.
- **Build-fail-fast over silent-default**: where a missing env var would silently break production (e.g., `EMAIL_FROM` unset), I prefer throwing at startup over falling back to `hello@example.com` (which Resend rejects with 422).

## Pre-final audit ritual

Before declaring any version of this skill final, I run a comprehensive grep audit for:
- Author-attribution leaks (specific names, brand names, domain literals, business IDs)
- The `whisper-user-benefits` slug (specific to one project; should never appear as example)
- Hardcoded brand colors, timezones, statutory citations, storage-key prefixes
- Trademarked terminology

The forbidden-strings list lives in this file's "Audit checklist" section below; it's the canonical reference.

## Audit checklist

Pattern list to grep against **the skill execution surface** before any final-version claim:

```
Drew|Whisperliminals|whisperlimin|RESTorative|restorativeco|getrestco|MintCRO|whisper-user-benefits|wl_analytics|wl_geo_country|wl_affiliate|#c9a96e|#f5f5f0|#0a0a0f|#3a3a3a|#9b9b9b|#eeeae0|mcp__scheduled-tasks__|America/Los_Angeles|RCW 19\.373|biz_1mHZqSNpmgJYkp|phc_va9w|wge0ywzwks
```

### Scope of the audit

The audit applies to files Claude actually reads during a skill run — these are the surfaces that pollute the user's session if they leak author-specific strings:

- `SKILL.md`
- `onboarding.md`
- All `stage-*.md` files
- All files under `_internal/`
- `examples/README.md`

The audit does NOT apply to:

- `LICENSE` — MIT copyright attribution requires a legal author name; this is required repo metadata, not a leak.
- `README.md` at the repo root — the GitHub landing page. Author attribution like `[@<github-handle>](...)` and the "Background" section's credit are intentional. The README is for humans browsing the repo, not for Claude during execution.
- `.gitignore`, `.git/`, anything outside the scope above.

### Expected matches inside this file

Two matches in this file (`_internal/claude-invariants.md`) are intentional and self-referential:
- The discussion-prose example mentioning `whisper-user-benefits` as the example forbidden slug.
- The regex pattern itself (it has to literally contain the strings it forbids).

These are not leaks. They're the rule definition.

### Verdict

Zero **unintended** matches across the audit-scope files = ready for final-version commit. Any other match = revisit + abstract before commit.
