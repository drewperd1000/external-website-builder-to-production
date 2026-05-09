# Stage 1: Project Scaffolding + Baseline Build

**Last verified: 2026-05-08.**

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) run all the scaffolding commands, write all the config files, and verify the baseline build works. The user does NOT run `npm install` / `git init` / `gh repo create` / etc. — I do that via my Bash tool. The user's only actions in this stage are: (1) confirming workspace location, (2) approving GitHub repo creation, (3) clicking through the GitHub OAuth flow if not already authenticated.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 1 needs:
- `project.slug` — used as the GitHub repo name and Railway project name. Auto-derived from `project.brand` if missing; user confirms.
- `project.brand` — used as default `<title>` and OG metadata. If missing, I prompt; this is blocking because every later stage references it.
- `project.site_url` — substituted into `.env.example`. If missing, I use the Railway staging URL placeholder and flag for Stage 4 to update.

I detect `styling.system` (`tailwind` / `plain` / `css-in-js`) at this stage and save it for Stage 8's ConsentBanner implementation choice.

## What I'm doing in this stage

Getting the source code into a fresh repo with a clean baseline build that runs locally. After this stage, the project's build pipeline is verified working — `npm install` + `npm run build` + `npm run dev` all succeed and a basic page renders at `http://localhost:5173` (or the framework's default port).

**Time**: 5–10 minutes of automated work.

## Prerequisites

- Stage 0 complete with `migration-punchlist.md` confirmed by the user.
- The source artifact is in hand (zip, repo, or files copied from production), or the user is starting fresh.
- The user has GitHub CLI authenticated (`gh auth status` returned green during onboarding's account check). If not, I install it and run `gh auth login` now — they click "Authorize" once in their browser.

## My execution sequence

### Step 1: I confirm workspace location with the user

```
Where should I put the new project on your machine?

  (a) Default: <user's home>/projects/<project-slug>
  (b) Inside an existing parent directory you specify
  (c) I'll just use the current directory

Which would you like?
```

If they pick (a), I create `~/projects/<slug>/` (or the platform equivalent). If (b), I use their path. If (c), I work in cwd.

### Step 2: I extract / clone / scaffold the source

Branching on origin platform (which I know from onboarding):

- **Zip from a website builder**: I extract to the workspace location.
- **Existing GitHub repo**: I run `git clone <url>` to the workspace location.
- **Hot-pulled HTML/CSS/JS from production**: I scaffold a fresh Vite + React project with `npm create vite@latest`, then copy the user's files in.
- **Starting fresh**: I scaffold a fresh Vite + React + TypeScript + Tailwind project.

### Step 3: I install dependencies and verify the baseline build

I run `npm install` and `npm run build`. If both succeed, I move on. If either fails, I diagnose:

- **`npm install` fails**: I check Node version (most modern stacks need ≥ 20). I look for platform-specific deps that don't install on the user's OS. If a different package manager's lockfile is present (yarn.lock, pnpm-lock.yaml), I report this to the user and ask which to standardize on.
- **`npm run build` fails**: TypeScript errors from the platform's auto-generated types — I either fix or annotate with `// @ts-expect-error` plus a comment. Missing files referenced by the platform's runtime — I cross-reference with the `migration-punchlist.md` from Stage 0; these get addressed in Stage 2. Vite plugin errors — I look for commented-out platform-specific plugins (Stage 0 already flagged these); the platform may have left a dev-only plugin import that crashes in production.

If I can't recover within 5 minutes, I stop and ask the user how they'd like to proceed (try a different Node version, accept some warnings, etc.).

### Step 3.5: I configure the `@/` path alias

Stages 6, 7, 8, 9, and 11 all import from `@/_core/...` (e.g., `import { readConsent } from "@/_core/consent"`). The alias must be configured in BOTH `tsconfig.json` (so TypeScript resolves it) AND `vite.config.ts` (so Vite's bundler resolves it at runtime). If either is missing, builds break with cryptic "Cannot find module '@/_core/consent'" errors.

**`tsconfig.json` — I add or merge into `compilerOptions`:**

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

If the file uses references (e.g., `tsconfig.app.json`), I add the same `baseUrl` + `paths` to the referenced config — Vite reads the leaf config, not the root.

**`vite.config.ts` — I add or merge the `resolve.alias` block:**

```ts
import path from "node:path";
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
});
```

I verify by running `npm run build` once after wiring both files. If it builds clean, the alias is wired correctly for everything later stages will throw at it.

If the user's source platform (Webflow / Framer / Readdy) shipped a `vite.config.ts` that already has a `resolve.alias` block, I merge mine in rather than overwriting. If they shipped a `jsconfig.json` instead of `tsconfig.json`, I add the same `baseUrl`/`paths` there.

### Step 4: I write `.gitignore`

If absent, I create at repo root:

```
node_modules
dist
out
build
.env
.env.local
.env.*.local
.skill-config.json
*.log
.DS_Store
desktop.ini
.vite
.cache
auto-imports.d.ts
*.tsbuildinfo
.idea
.vscode
*.swp
```

If a `.gitignore` exists from the source platform, I audit it and add anything missing. Specifically I make sure `node_modules`, `.env`, `.skill-config.json`, and `desktop.ini` (Windows + cloud-sync corruption risk) are all listed.

### Step 5: I write `.env.example`

Created at repo root with EVERY env var that later stages will set, using safe placeholder values. This is the future-proofing against the silent-Vite-build trap (SKILL.md invariant #1).

```bash
# Build-time vars (Vite bakes into bundle — must be set BEFORE every build)
VITE_POSTHOG_KEY=phc_xxxxxxxxxxxx
VITE_POSTHOG_HOST=https://<your-domain>/<your-proxy-slug>
VITE_CLARITY_PROJECT_ID=<your-clarity-id>

# Runtime vars (server-side; set on Railway service)
RESEND_API_KEY=re_xxxxxxxxxxxx
EMAIL_FROM=<Brand Name> <hello@<your-domain>>
EMAIL_NOTIFY_TO=hello@<your-domain>
SITE_URL=https://<your-domain>
APP_URL=https://app.<your-domain>
PORT=3000

# Optional: timezone for date-formatting in events (Stage 6)
# Any IANA tz like "America/New_York", "Europe/London", "Asia/Tokyo".
# Defaults to UTC if unset — safe everywhere; only override if dashboards
# need wall-clock days aligned to a specific region.
DEPLOY_TIMEZONE=UTC

# Optional: webhook integration (Stage 9)
WHOP_WEBHOOK_SECRET=whsec_xxxxxxxxxxxx
WHOP_API_KEY=xxxxxxxxxxxx
POSTHOG_PROJECT_API_KEY=phc_xxxxxxxxxxxx
```

I substitute `<your-domain>` with the user's actual domain at write time. Placeholders for keys remain `xxxxxxxxxxxx` style — the user pastes real values later (or I fetch them from a vendor's API where possible).

### Step 6: I initialize git (if not already a repo)

I run `git init`, then check if `git config user.name` and `user.email` are set globally. If yes, I use those. If not, I ask the user once and then set them. I make the initial commit:

```
git add .
git commit -m "Initial commit: <platform>-generated baseline"
```

### Step 7: I create the GitHub remote

I run `gh repo create <slug> --private --source=. --remote=origin --push` (or public if the user prefers). If `gh` isn't authenticated, I prompt the user to run `gh auth login` once — they click "Authorize" in their browser. After that, every subsequent `gh` call is autonomous.

Stage 4 (Railway) will connect to this GitHub repo for continuous deploys.

### Step 8: I detect the framework

Most modern marketing sites are Vite + React + TypeScript + Tailwind. If the project is something else, I auto-detect from `package.json` and adapt:

| Framework | Build output | Dev port | Stage 3/4 changes I apply |
|---|---|---|---|
| **Vite + React** | `dist/` (Vite default) | 5173 | Default — Stages 3/4 assume this |
| **Next.js** | `.next/` (server) | 3000 | Stage 3 may not need an Express layer (Next has its own server). PostHog reverse proxy goes in `middleware.ts` or `pages/api/` instead. |
| **Astro** | `dist/` | 4321 | Stage 3 Express works; Astro's adapter setup may add complexity |
| **SvelteKit** | `.svelte-kit/output/` | 5173 | Stage 3 needs the Node adapter; PostHog proxy in `hooks.server.ts` |
| **Remix** | `build/` | 3000 | Built-in Express, integrate PostHog proxy in the Express server |

**Important**: Vite's default `outDir` is `dist/`. Some templates (notably some Readdy exports) override this to `out/`. I check `vite.config.ts` and report what the actual output directory is — Stage 3's `server.js` template needs to point at the right one.

If the framework is a non-React-non-Vite stack, I document the adaptation in this stage file's changelog as I go.

### Step 8b: I detect the styling system

Stage 8's ConsentBanner has two implementations (Tailwind-class and inline-styles). I detect which one to ship by inspecting the project:

| Signal | Result |
|---|---|
| `tailwindcss` in `package.json` `dependencies` or `devDependencies` AND `tailwind.config.{js,ts,cjs,mjs}` exists | `styling.system = "tailwind"` |
| `@emotion/*` or `styled-components` in deps | `styling.system = "css-in-js"` (treat as `plain` for ConsentBanner — inline-styles version is the safest baseline) |
| Plain `.css` files imported with `import "./foo.css"` and no Tailwind detected | `styling.system = "plain"` |
| Webflow export (Stage 0 flagged) | `styling.system = "plain"` (Webflow's own CSS handles the rest of the site; the banner is independent) |
| Framer export | `styling.system = "plain"` |

I save `styling.system` to `<project>/.skill-config.json` and Stage 8 reads it to pick the right ConsentBanner implementation.

**If the user has no styling system at all** (rare; would be a hand-rolled HTML site), I offer to install Tailwind in this stage (~90 seconds) so the rest of the skill can lean on its conventions. The user accepts/declines; declining means inline-styles everywhere and a slightly less polished aesthetic.

## Common platform-specific bugs I fix automatically

If Stage 0 identified one of these origin platforms, I apply the relevant fixes during Step 4 / Step 5 above without extra prompting.

### Readdy-origin fixes

#### Bug R1 — Vite port hardcoded

Readdy's `vite.config.ts` ships with `port: 3000` literal. I change to read from `process.env.PORT`:

```ts
server: {
  port: process.env.PORT ? Number(process.env.PORT) : 3000,
  host: "0.0.0.0",
}
```

Without this, Railway's `PORT` env var is ignored at dev preview time.

#### Bug R2 — Missing scroll-to-top on route change

Readdy uses react-router-dom without the standard scroll-restoration wrapper. I add the standard `useEffect([pathname], () => window.scrollTo(0,0))` to `src/router/index.ts` (or wherever `useRoutes` is called). This same `useEffect` is where the manual `posthog.capture('$pageview', ...)` will go in Stage 6.

#### Bug R3 — Empty `<title>` + placeholder favicon

Readdy's `index.html` ships with empty title + the Vite default `vite.svg` favicon. I update with:
- The brand name and tagline as `<title>` (using values from onboarding)
- A real favicon path (or remove the placeholder line if `public/favicon.ico` exists)
- Full OpenGraph + Twitter card metadata (required for affiliate URL unfurling at Stage 10):

```html
<meta name="description" content="<from onboarding's brand description>">
<meta property="og:title" content="<from onboarding>">
<meta property="og:description" content="<from onboarding>">
<meta property="og:image" content="https://<your-domain>/og-image.png">
<meta property="og:url" content="https://<your-domain>/">
<meta property="og:type" content="website">
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="<from onboarding>">
<meta name="twitter:description" content="<from onboarding>">
<meta name="twitter:image" content="https://<your-domain>/og-image.png">
```

If the user doesn't have an OG image yet, I leave a placeholder URL and flag it in the punchlist for Stage 12.

### Webflow-origin fixes

- **jQuery dependency**: Webflow exports include `webflow.js` which depends on jQuery. I leave both in place by default — they work fine. If the user wants to rewrite the interactive parts (animations, dropdowns) in vanilla JS / React, I flag it as Stage 12 polish.
- **Form action URLs**: Webflow forms POST to `https://webflow.com/api/v1/form/...`. I note these in the punchlist; Stage 5 replaces them.
- **Webflow-specific CSS classes** (`w-form`, `w-button`, `w-container`): safe to keep (just CSS).

### Framer-origin fixes

- **Asset CDN**: `framerusercontent.com` URLs throughout. Stage 0 catalogued these; Stage 2 self-hosts them.
- **Font CDN**: Framer injects from `fonts.framer.app`. I migrate to Google Fonts or self-hosted in Stage 2.
- **Code components**: Framer's "code components" export as React components but may reference internal Framer APIs. I audit each one; some are pure React (keep), some need rewriting (flag in punchlist).

## Verification (autonomous)

I run all of these myself before declaring Stage 1 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| Build succeeds | `npm run build` | Output directory populated, no errors |
| Dev server starts cleanly | `npm run dev` (then I curl localhost) | HTTP 200 returned |
| Type-check passes (if TypeScript) | `npm run type-check` (or `npx tsc --noEmit`) | Zero errors |
| Lint passes (if linter configured) | `npm run lint` | Zero errors (warnings OK to defer) |
| Git is clean | `git status` | Nothing to commit |

I report results to the user as a brief summary:

```
✅ Stage 1 complete:
   • Project at ~/projects/mayasconsulting-marketing/
   • Pushed to github.com/maya-username/mayasconsulting-marketing
   • npm run build: success (output in dist/, 247 KB gzipped)
   • npm run dev: starts on port 5173, returns 200
   • Type-check: 0 errors
   • Git status: clean

Ready for Stage 2 (platform migration)?
```

If any verification fails, I debug within this stage and don't advance.

## Common gotchas I handle automatically

- **`npm run check` vs `npm run type-check`**: some projects use `check`. I verify the actual script name in `package.json` before running.
- **`--legacy-peer-deps` may be required** if the source platform pinned old peer deps (some AI-generator outputs hit this with React 19 + older plugin versions). I detect the failure mode and retry with `--legacy-peer-deps`, documenting in `package.json` notes.
- **Cross-platform shell**: I use my Bash tool which abstracts away Windows/macOS/Linux differences. The user doesn't need to know about PowerShell vs bash distinctions.

## Outputs

After Stage 1:
- A clean repo on GitHub with one initial commit
- `.gitignore` + `.env.example` at root with appropriate placeholders
- Verified working baseline build
- All Stage 0 platform-specific scaffolding bugs fixed (per-platform)
- A summary report ready for the user

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. |
| 2026-05-08 | Reframed for non-coder audience: every "Run X" / "Edit Y" instruction now first-person Claude voice ("I run...", "I write..."). Removed author-specific workspace path conventions. Cleaned `.env.example` template to use generic `<your-domain>` placeholder + standard `phc_xxxxxxxxxxxx` key shape. Added explicit note that Vite's default outDir is `dist/` (some Readdy templates override to `out/`); Stage 3's server.js needs to point at whichever is actually present. Verification section reframed from user-runs-bash to "I run these checks; here's the summary I report." Stripped author-specific git-worktree gotcha. |
