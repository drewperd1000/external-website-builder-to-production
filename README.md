# external-website-builder-to-production

A Claude Code skill that takes a website built somewhere else — Webflow, Framer, Wix Studio, Readdy, Bubble, an AI generator, or your own custom code — and ships it to fully-instrumented production on Railway with analytics, email, consent banner, and (optional) memberships and affiliate tracking.

> **Who this is for**: founders, marketers, and developers who want a real production stack without having to learn the operational stack themselves. The skill is designed to be Claude-driven — Claude executes the technical work, you make the business decisions.

## What you get when this skill runs end-to-end

- **A Railway-deployed Vite + React + Express site** at your custom domain
- **PostHog analytics** with a same-origin reverse proxy (recovers 10–25% of events that ad-blockers would otherwise drop), `loaded`-callback init pattern (no race), and build-time deploy + variant identifiers attached to every event
- **Microsoft Clarity** heatmaps + session replay alongside PostHog (the unlimited free-fallback for replay capacity past PostHog's free-tier cap of 5k desktop / 2.5k mobile sessions per month)
- **Resend** transactional email behind a swappable transport abstraction (welcome emails, contact-form notifications, autoresponders) with DKIM/SPF wired
- **A geo-aware cookie consent banner** that adapts to your privacy posture (strict / standard / light) with three-layer Clarity content masking
- **Optional: payments + memberships** via Stripe (default), Whop (recommended for content/community/course businesses), or Skool / Lemonsqueezy / Paddle / Gumroad — with HMAC-verified webhooks forwarding subscription events server-side to PostHog
- **Optional: affiliate program** with multi-layer attribution surviving cookie deletion, Safari ITP, AND ad-platform username changes
- **Optional: variant tracking** for sequential CRO iteration with 6+ pre-built PostHog dashboards
- **A pre-launch checklist** that catches the silent-failure modes before paid traffic hits the site

The skill ships 13 stages (0 through 12), each with a "Last verified" date and re-research instructions for when vendor APIs evolve.

## Prerequisites

- **Claude Code** ([install](https://docs.anthropic.com/claude-code))
- **Node.js ≥ 20** ([install](https://nodejs.org/))
- **Git** ([install](https://git-scm.com/))
- **GitHub CLI** (`gh`) ([install](https://cli.github.com/))
- A **Railway** account (free tier works for getting started)
- Optional accounts created during the skill run: PostHog, Microsoft Clarity, Resend, Cloudflare, Stripe / Whop / etc.

The skill walks you through signing up at each vendor when it's time. You don't need them all upfront.

## Install

### Option 1 — Personal use (most common)

Clone into your Claude Code skills directory:

```bash
# macOS / Linux
git clone https://github.com/drewperd1000/external-website-builder-to-production.git \
  ~/.claude/skills/external-website-builder-to-production
```

```powershell
# Windows PowerShell
git clone https://github.com/drewperd1000/external-website-builder-to-production.git `
  $env:USERPROFILE\.claude\skills\external-website-builder-to-production
```

```bash
# Windows Git Bash
git clone https://github.com/drewperd1000/external-website-builder-to-production.git \
  "$USERPROFILE/.claude/skills/external-website-builder-to-production"
```

That's it. Next time you start a Claude Code session in any project, the skill is available.

### Option 2 — Workspace-local (if you want to track your own modifications in the project repo)

Clone into your project's `.claude/skills/` directory instead:

```bash
git clone https://github.com/drewperd1000/external-website-builder-to-production.git \
  /path/to/your/project/.claude/skills/external-website-builder-to-production
```

This isolates the skill to that project. Useful if you're forking it.

## How to use it

Start a Claude Code session in the project directory where the website code will live. Tell Claude what you're trying to do — any of these phrasings (or similar) will trigger the skill:

```
I have a Readdy zip and want to deploy it to production.

Set up production for this Webflow export.

Build the production stack for this landing page.

Wire PostHog reverse proxy + cookie banner on a new site.
```

Claude reads `SKILL.md`, then the appropriate stage files, and starts the onboarding flow. Onboarding asks you a few questions about how technical you are (`non_coder` / `developer` / `expert`), how you want to work (everything upfront vs. phase-by-phase), and a handful of project details. Then Claude executes.

## What's in this repo

```
SKILL.md                        ← skill entry point; Claude reads this first
onboarding.md                   ← onboarding flow (modes, expertise, project questions)
stage-0-discovery.md            ← source-platform dependency discovery
stage-1-scaffolding.md          ← clean repo + baseline build
stage-2-platform-migration.md   ← migrate off the source platform's infrastructure
stage-3-express-server.md       ← Express server (proxy + webhooks + forms + redirects)
stage-4-railway.md              ← Railway deploy with env-vars-before-build
stage-5-email-resend.md         ← transactional email + form endpoints
stage-6-posthog.md              ← PostHog with reverse proxy + loaded callback
stage-7-clarity.md              ← Microsoft Clarity with three-layer masking
stage-8-privacy-consent.md      ← consent state machine + geo-aware banner
stage-9-membership-optional.md  ← Stripe / Whop / Skool / etc. webhook handler
stage-10-affiliate-optional.md  ← multi-layer attribution + branded short URLs
stage-11-variants-and-dashboards.md  ← variant system + 6+ PostHog dashboards
stage-12-launch-checks.md       ← pre-launch verification with severity buckets

_internal/
  claude-invariants.md          ← technical rules Claude follows when writing code
  local-preview.md              ← local dev verification patterns
  reference-railway-automation.md  ← Railway MCP / CLI / GraphQL deep dive
  reference-cloudflare-dns.md   ← Cloudflare DNS patterns + cdn-cgi/trace
  reference-object-storage.md   ← Railway Buckets vs Backblaze B2 decision tree

examples/
  README.md                     ← pointers to where each canonical code pattern lives
```

## Updating

This skill evolves as vendor APIs change. To pull updates:

```bash
cd ~/.claude/skills/external-website-builder-to-production
git pull
```

The bottom of each stage file has a Changelog section dated by when it last shifted.

## Voice + automation contract

The skill follows a strict voice convention so you always know who's doing what:

- **First-person ("I run...", "I write...")** — Claude actually executes via tools (Bash, Edit/Write, MCP)
- **Second-person ("Please open...", "Click here...")** — you must take the action (sign up at a vendor, click in a vendor's no-API dashboard, OAuth approve)
- **Mixed ("Paste the key here; I'll save it...")** — you provide input, Claude processes it

If you ever see Claude claim "I'll do X" but X is something it can't actually do (open arbitrary browser tabs without a Chrome MCP, sign up at vendors as you, etc.), that's a bug — flag it.

## Privacy + safety

- **No vendor accounts are created without your consent.** When the skill needs you to sign up somewhere, it pauses and asks.
- **Secrets stay out of the chat.** API keys are pasted into project-local `.secrets/<vendor>-key.txt` files (gitignored) and shell-substituted into Railway env vars without entering Claude's context.
- **The `_internal/claude-invariants.md` file** documents the technical rules Claude follows when writing code (including a fail-fast-on-missing-env-vars rule that catches the most common silent-production-break).

## Compatibility

| Item | Verified |
|---|---|
| Last vendor-API verification | 2026-05-08 (see "Last verified" lines on each stage file) |
| Source platforms | Readdy, Webflow, Framer, Wix Studio, Bubble, custom code, AI generators (Lovable, v0, Bolt) |
| Frameworks | Vite + React + TypeScript (default), with adaptation notes for Next.js, Astro, SvelteKit, Remix |
| Hosts | Railway (canonical); the architectural patterns adapt to Render, Fly.io, AWS App Runner |
| Operating systems | macOS, Linux, Windows (Claude Code Bash tool abstracts shell differences) |

## Contributing

Issues + pull requests welcome. The skill is designed to be a living document — each stage file's Changelog is appended in place when patterns evolve.

If you find a vendor API has shifted (e.g., Railway adds a tool the skill doesn't yet use, PostHog's reverse-proxy mechanics change), open an issue with:
- Which file + stage
- What the doc currently says
- What the vendor's docs now say
- Any error messages or symptoms you hit

## License

MIT — see [LICENSE](LICENSE) for the full text. Use it freely, fork it, adapt it. Attribution appreciated but not required.

## Background

This skill was built by [@drewperd1000](https://github.com/drewperd1000) to consolidate the production-stack patterns learned from shipping marketing sites and apps in 2026. The patterns are vendor-current as of 2026-05; the skill's self-research instruction tells Claude to re-verify the externally-versioned facts (MCP tool names, CLI verbs, pricing tiers) before any stage older than 60 days.
