---
name: external-website-builder-to-production
description: Use when bringing an externally-built website (Readdy zip, Webflow/Framer export, Wix Studio, Bubble, custom code, AI generator output, etc.) to fully-instrumented production on Railway. Triggers on phrases like "deploy this site to production", "set up the analytics stack on a new site", "wire PostHog reverse proxy", "bring my Readdy/Webflow/Framer site to production", "set up Resend on Railway", "add cookie banner with geo-detection", "build the production stack for this landing page", "configure the variant tracking system", "set up affiliate tracking on the new site", or any combination of Railway + PostHog + Clarity + Resend + cookie banner + variant tracking on a new site.
---

# External Website Builder to Production

**Living skill — last updated 2026-05-08.** Vendor automation surfaces (Railway MCP, PostHog MCP, Clarity MCP, Resend API) evolve fast. Before I apply any externally-versioned fact in this skill, I check the "Last verified" line on the relevant stage file. If it's older than 60 days, I re-research by web-searching `<vendor> MCP server <year>` and reading the current README, then append findings to the Changelog so future sessions inherit them.

---

## ⚠️ Automation contract — read first; re-read at the start of every stage

**This is a Claude-driven skill.** When the user invokes me with this skill loaded, I (Claude) execute the technical work. The user makes decisions and provides a small number of inputs at moments where my tooling cannot act on their behalf (account signups, dashboard clicks at vendors that have no API, secret pastes).

For every action this skill describes, the default behavior is:

| What | Who does it | Notes |
|---|---|---|
| Run shell commands (`npm`, `git`, `gh`, `railway`, `curl`, `dig`, etc.) | I do — via Bash | After one-time `gh auth login` / `railway login` browser approve |
| Create or edit code/config files | I do — via Edit/Write | All `server.js` / `vite.config.ts` / `package.json` / etc. |
| Verify outputs (network requests, console logs, screenshots, build artifacts) | I do — via Preview MCP / Chrome MCP / Bash | Falls back to user-action if MCPs unavailable |
| Open URLs in user's browser (signup pages, dashboards) | I do via Chrome MCP **if connected**; otherwise user opens manually | I always provide the URL |
| Sign up at vendor (Resend, PostHog, Clarity, etc.) | **User does** — Claude can't be them at the identity layer | I provide URL, click path, save the API key the user pastes back |
| Approve OAuth browser flows | User clicks "Approve" once per service | Subsequent operations on that vendor are autonomous (modulo current Claude Code OAuth-persistence bugs that occasionally require re-approve) |
| Click through dashboard UIs (vendors with no API for the action) | **User clicks** — I provide exact step-by-step click paths | Examples: Whop product setup, Clarity masking config |
| Add DNS records | I do via Cloudflare API **if user granted a CF API token at onboarding**; otherwise user pastes records into their DNS provider | Asked once at onboarding; unlocks Stages 4 + 5 + 10 autonomously |
| Provide secrets / API keys | User pastes once into a project-local secrets file; I read + use without displaying | Pattern documented in stage-4-railway.md Step 3b |
| Make business / brand / legal decisions | User decides; I ask plain-language questions in their vocabulary | Pricing, privacy posture, brand voice, etc. |

### Voice rule

- **First-person ("I run...", "I write...")** — only when I actually execute via tools (Bash, Edit/Write, MCP)
- **Second-person ("Please open...", "Click here...")** — only when the user must take the action (signup, dashboard clicks at no-API vendors, OAuth approves)
- **Mixed ("Paste the key here; I'll save it...")** — when the user provides input I then process

If a stage describes "I'll do X" but X is something Claude cannot actually do (open the user's web browser to sign them up at a vendor, click in a vendor's account-setup wizard), that's a bug — flag it and I fix.

**Re-grounding rule**: at the start of every stage, I re-read this Automation Contract before executing. If I catch myself describing what the user should do in a step that I could actually automate via my tools, I stop and reframe — and vice versa, if I catch myself claiming I'll do something my tools can't, I stop and reframe to honest user-action language.

---

## User expertise mode

I behave differently based on the user's comfort level. Onboarding asks the user which mode fits them (or auto-detects from their first few messages).

| Mode | When | What I do differently |
|---|---|---|
| **non_coder** (default) | The user has never opened a terminal, hasn't used git/npm, hasn't edited code beyond website-builder UIs. Marketers, founders, designers using Claude Code as their dev partner. | I never show raw code unless asked. I never ask the user to type shell commands. I explain every dev term in plain language at first use. I confirm major actions before running them ("I'm about to deploy. Ready?"). I default to yes/no or pick-from-list questions. |
| **developer** | The user has shipped a few projects, knows git, has used at least one cloud host, can read a `.tsx` file. | I skip beginner explanations. I show diffs when relevant. I run commands without preamble. Questions can include moderate jargon. |
| **expert** | The user has shipped many sites, runs PostHog routinely, knows Cloudflare proxy modes cold, uses Claude Code daily. | Minimum prose. Maximum throughput. I auto-pick defaults; user vetoes. I can answer "explain less" mid-stage. |

The mode lives in the saved skill config as `user_expertise: "non_coder" | "developer" | "expert"`. If unset, I default to `non_coder` and adjust if the user proves more advanced.

---

## Onboarding mode

Two ways for the user to start. Onboarding asks which fits — and I explain both options in plain language so the user can pick the right one for their situation.

### Mode A — `all_upfront`

**What happens**: I ask every question I'll need across the entire build, all in one sitting. Then the user steps away. I run all 13 stages autonomously, pausing only for OAuth approvals (3–4 browser clicks) and any secret-paste moments where the user has to copy a key from a vendor's dashboard.

**Benefits**:
- The user spends ~30–60 minutes upfront making decisions, then can walk away for the rest. Their site builds while they do other work.
- No mid-day interruptions. Every decision is captured at one moment when the user's attention is allocated to this project.
- Total wall-clock time is shorter because async waits (DNS propagation, SSL cert issuance, build queues) overlap.
- If the user's schedule is "I have an evening free" or "I want this set up overnight" — this is the right pick.

**Tradeoffs**:
- The user has to make decisions about things they may not yet have full context on. (Mitigation: I default to recommended values for anything they're not sure about, and they can change later.)
- If the user underestimates how long they'll need the questionnaire to take, they may bail out partway and end up in `phase_by_phase` mode anyway. I check at minute 30: "Want to switch to phase-by-phase?"

### Mode B — `phase_by_phase`

**What happens**: I ask only the minimum questions needed to start (~10 minutes' worth — basically: brand name, domain, where the source code lives). Then I begin Stage 0 and work through stages, asking the questions each stage needs as we hit them.

**Benefits**:
- The user sees real progress (commits, deployments, working pages) within the first session.
- Decisions arrive in context — when I ask about the Resend sending domain in Stage 5, the user is already thinking about email and the question makes more sense than it would have in an upfront block.
- No big upfront cognitive block. Easier to start when the user has limited focus or is uncertain about some choices.
- The user can pause between stages and come back tomorrow without having "wasted" any setup.
- If the user's schedule is "I'll work on this in chunks across the week" or "I want to see something working before I keep investing" — this is the right pick.

**Tradeoffs**:
- More interruptions over the next few hours/days. Every stage has a "wait, I need to ask you something" moment.
- The user has to remain reachable (or at least responsive within a reasonable window) for the build to complete.
- Total wall-clock time is longer because async waits don't overlap as efficiently — Resend DKIM verification might block Stage 5 because we didn't start it during onboarding.

### Default

If the user is unsure which fits, I default to **`phase_by_phase`** — lower risk of bailing partway, and the user can always upgrade to `all_upfront` mode later by saying "go ahead and set up everything you can; I'll be back in an hour."

The mode lives in the saved skill config as `onboarding_mode: "all_upfront" | "phase_by_phase"`. The user can switch modes mid-flight by saying "switch to all_upfront" / "let me answer everything now" / "stop asking me — pick defaults" — I update the config and proceed accordingly.

---

## Placeholder conventions used throughout this skill

The skill uses these placeholders in examples and templates. **I substitute these with the user's real values during execution. I never ship a literal placeholder to a user's project.**

| Placeholder | Real-value shape |
|---|---|
| `Maya's Consulting` | The user's brand name |
| `mayasconsulting.com` | The user's primary domain |
| `app.mayasconsulting.com` | The user's app subdomain (if applicable) |
| `mayas.link` | The user's branded short-URL domain (if applicable) |
| `hello@mayasconsulting.com` | The user's notification email address |
| `phc_xxxxxxxxxxxx` | PostHog Project API Key |
| `<your-clarity-id>` | Clarity Project ID |
| `<your-proxy-slug>` | The chosen PostHog reverse-proxy path slug |
| `biz_xxxxxx` | Whop business ID (if Whop) |
| `plan_xxxxx` | Whop plan ID (if Whop) |
| `<your-domain>` / `<your-X>` | Generic placeholder for any user-specific value |

These are the canonical placeholder names. When I see one of these in an example, I replace with the user's actual value before applying.

---

## What this skill does

Takes a website built somewhere else — a low-code platform export, an AI site generator zip, a Figma → React handoff, custom code — and produces a fully-instrumented production deployment with:

- **Railway hosting** with proper env-var-before-build discipline + custom domain
- **PostHog product analytics** with same-origin reverse proxy (recovers 10–25% of events lost to ad-blocker fingerprinting)
- **Microsoft Clarity** heatmaps + session replay with PostHog identity bridging
- **Resend** transactional email behind a swappable transport abstraction (so future SES/Postmark/Mailgun migrations are mechanical)
- **Geo-aware consent banner** with three privacy posture options plus customization
- **Variant tracking system** (global cohort + per-page lineage) supporting sequential single-page CRO methodology, with deploy identifiers attached to every event for clean cohort analysis
- **Optional**: Whop / Skool / etc. membership integration with server-side webhook → PostHog forwarding
- **Optional**: multi-layer affiliate attribution surviving cookie deletion AND ad-platform username changes (verified gotcha — see Stage 10)

## When to use this skill

Direct triggers from the user:
- "Set up production for [a website]"
- "Deploy this Readdy/Webflow/Framer/Wix/etc. site to Railway"
- "Wire analytics on the new marketing site"
- "Add a cookie banner that's GDPR-compliant"
- "Set up affiliate tracking for the marketing site"
- ANY combination of: Railway + PostHog + Clarity + Resend + cookie banner + variant tracking on a NEW site

## When NOT to use this skill

- **The site is already live and the user just wants to fix one integration** → I open the relevant `stage-N-*.md` file directly. Don't run the whole flow.
- **Pure backend service with no marketing surface** → most stages don't apply; cherry-pick.

## Process — onboarding + 13 stages

I execute the stages in order. Each stage outputs artifacts the next stage depends on, so I don't skip ahead.

**Always start with [onboarding.md](onboarding.md)** — it captures expertise mode, onboarding mode, account/tool prereqs, and the decisions later stages need. At the end of onboarding I save the answers to a project-local `.skill-config.json` (gitignored) so future sessions pick up where they left off.

| Stage | What | File | Required? |
|---|---|---|---|
| **Onboarding** | Mode + expertise + accounts + decision capture | [onboarding.md](onboarding.md) | ✅ Always — start here |
| 0 | Code-discovery + migration punchlist | [stage-0-discovery.md](stage-0-discovery.md) | ✅ Always |
| 1 | Scaffold project + baseline build | [stage-1-scaffolding.md](stage-1-scaffolding.md) | ✅ Always |
| 2 | Migrate platform-specific dependencies | [stage-2-platform-migration.md](stage-2-platform-migration.md) | ✅ Always (if any from Stage 0) |
| 3 | Add Express server | [stage-3-express-server.md](stage-3-express-server.md) | ✅ Always |
| 4 | Railway service deploy (env BEFORE build) | [stage-4-railway.md](stage-4-railway.md) | ✅ Always |
| 5 | Email provider | [stage-5-email-resend.md](stage-5-email-resend.md) | If forms or transactional emails exist |
| 6 | PostHog setup with reverse proxy | [stage-6-posthog.md](stage-6-posthog.md) | ✅ Always |
| 7 | Clarity setup | [stage-7-clarity.md](stage-7-clarity.md) | Strongly recommended |
| 8 | Privacy + consent | [stage-8-privacy-consent.md](stage-8-privacy-consent.md) | ✅ Always |
| 9 | Membership platform | [stage-9-membership-optional.md](stage-9-membership-optional.md) | If selling subscriptions/memberships |
| 10 | Affiliate tracking | [stage-10-affiliate-optional.md](stage-10-affiliate-optional.md) | If running an affiliate program |
| 11 | Variant system + dashboards | [stage-11-variants-and-dashboards.md](stage-11-variants-and-dashboards.md) | Strongly recommended |
| 12 | Pre-launch checks | [stage-12-launch-checks.md](stage-12-launch-checks.md) | ✅ Always |

Plus three cross-cutting reference files (kept under `_internal/` because they're Claude's working notes — not user-facing):

- **[`_internal/local-preview.md`](_internal/local-preview.md)** — `npm run dev` vs `npm run start`, `.env.local` for prod-parity, autonomous-verification via Preview MCP / Chrome MCP, multi-device LAN access, Lighthouse audits.
- **[`_internal/reference-railway-automation.md`](_internal/reference-railway-automation.md)** — Railway's MCP / CLI / GraphQL surfaces.
- **[`_internal/reference-cloudflare-dns.md`](_internal/reference-cloudflare-dns.md)** — `cdn-cgi/trace` geo detection, proxy vs DNS-only decisions, Cloudflare Error 1000 explainer, SPF stacking, Redirect Rules.
- **[`_internal/reference-object-storage.md`](_internal/reference-object-storage.md)** — Railway Buckets vs Backblaze B2 decision framework, the public-URL caching gotcha, and the rule for defaulting to Backblaze when the public-vs-private question is uncertain.

## Technical invariants Claude follows when writing code

There are six silent-failure modes Claude needs to avoid when writing the code for this skill (Vite build-time bake, proxy ordering, super-property timing, vendored static, Clarity masking layers, privacy-posture lock-in). The user doesn't need to memorize these — what matters is the outcome: analytics work, masking works, consent banner shows in the right places.

The full technical detail lives at **[`_internal/claude-invariants.md`](_internal/claude-invariants.md)**, which Claude reads and follows. The user can review it if they want to understand exactly what Claude is doing under the hood, but it's not required reading.

## Process loop within each stage

For every stage I execute:
1. **Re-read the Automation Contract** at the top of this file (combat drift to "tell user what to do").
2. **Read the stage file** before doing anything.
3. **Run the required-values check** (see "Required-values check pattern" below).
4. **Follow it in order** — instructions are sequenced because of inter-step dependencies.
5. **Verify the output** matches the stage's "Verification" section using my own tools.
6. **Stop on verification failure** — I debug within the stage. Do not advance.
7. **Show the user a plain-language summary** of what I did and what they need to confirm before next stage.

## Required-values check pattern (universal — runs at start of every stage)

Onboarding may not have captured every value a stage needs. Reasons:
- The user picked `phase_by_phase` mode (only 4 essential questions answered upfront)
- The user is using a saved config from an older skill version that didn't include a newer field (e.g., `brand_prefix`, `cloudflare_token`, `storage.use_case`, `analytics.stack`, `styling.system`, `hosting.railway.service_id`)
- A stage was previously skipped and is now being run

**The pattern, executed at the start of every stage**:

1. I read `<project>/.skill-config.json` and check for the values this specific stage needs (each stage's file lists these in its "Required-values check" preamble — Stage 8 is the canonical example).
2. I categorize missing values as either **blocking** (this stage cannot meaningfully produce output without them) or **deferrable** (this stage can do partial work; the value is needed before a specific later step within the stage).
3. **For blocking-missing values**: I prompt the user now in plain language, explaining what the value is and why it matters. I provide an auto-proposal where one is sensible (e.g., a brand prefix derived from the slug). I save the user's answer to skill config so future stages and future runs don't ask again.
4. **For deferrable-missing values**: I do the work that doesn't depend on them in parallel while asking the user to fetch / decide. The user's clarification arrives, I integrate it, and forward progress is unblocked. The classic case: "I need your Resend API key, but I can finish writing the email module's transport abstraction while you grab it from the dashboard."

**Why this exists**: when a saved config is incomplete, the right behavior is to ask narrowly for what's missing — NOT to bail out, and NOT to silently substitute placeholder garbage that breaks at runtime. The fallback prompt is small and specific; the user answers in seconds; the stage proceeds.

**My discipline**: if I find myself writing code that references a config value I haven't verified is set, I stop and re-run the required-values check. The Stage 5 `EMAIL_FROM` invariant in `_internal/claude-invariants.md` exists because exactly this kind of silent placeholder substitution shipped to production once and broke immediately.

## Common rationalizations to refuse

| Thought | Reality |
|---|---|
| "I'll just set the Vite env vars later" | Vite bakes at build. Later = silent prod outage until next deploy. |
| "I'll skip the reverse proxy for v1" | 10–25% of events are lost to ad-blockers from day one. Migration cost only grows with traffic. |
| "I'll add the consent banner before launch" | Once analytics is live without it, retention rules may force discarding data. Banner first. |
| "I'll use the standard `i.posthog.com` URL" | Adblocker fingerprints. Same-origin from day 1. |
| "One giant useEffect for pageview tracking is simpler" | Races posthog-js init. Loaded callback is the documented pattern. |
| "Whop's webhook payload has the affiliate, right?" | No (verified 2026-04-28). Affiliate attribution must be DB-mediated. |
| "I'll just hardcode the domain in two places" | When DNS migrates, every hardcoded reference surfaces the hard way. Use `APP_URL` env var. |
| "Skipping Stage 0 — I know what's in this codebase" | Stage 0 surfaces things confident grepping misses (commented-out plugins, hidden form data attrs, FTC-vulnerable AI faces). 30 minutes saves a launch-blocker incident. |
| "The user can run this command themselves" | Default is no — I run it. Only ask the user to type a command if I genuinely cannot (e.g., interactive password prompt I can't proxy). |

## Self-research instruction

Before starting **Stage 4** (Railway), **Stage 6** (PostHog), or **Stage 7** (Clarity):

1. I web-search `<vendor> MCP server <current year>` and read the current README.
2. I compare against the "Last verified" line on the corresponding stage file.
3. If the auth model or capability surface has changed, I USE the new affordances and append a dated entry to the Changelog at the bottom of the relevant stage file.

The vendor landscape moves faster than this skill can. Stage files are written to be REPLACED in place, not patched defensively.

## Pre-final audit checkpoint (before any "skill is shipped" claim)

Whenever this skill is materially edited, I run the audit procedure documented in [`_internal/claude-invariants.md`](_internal/claude-invariants.md) before declaring the edit complete. This is non-optional — past versions of the skill leaked author-specific brand strings and storage-key prefixes despite passing surface-level review. The forbidden-strings pattern list in that file is the canonical reference. **Zero matches across the entire skill = ready to commit. Any match = revisit and abstract before committing.**

I run the audit specifically when:
- A new stage file is added or substantially rewritten.
- Code samples have been pasted from a real project (high risk of literal domains, key prefixes, brand colors, statutory citations).
- Voice has been edited (drift between first-person Claude actions and second-person user actions is the source of "I'll do X" fictions where Claude can't actually do X).
- The Automation Contract has been touched.

## Companion skills (optional; pull in when their value matches the moment)

This skill handles deployment + infrastructure + instrumentation. It deliberately does NOT prescribe a visual aesthetic, copywriting voice, or accessibility methodology. If the user has design-, marketing-, or quality-focused skills installed in their Claude Code environment, I can invoke them at moments where their specific value applies. Full mapping in the [README's "Companion skills" section](README.md#companion-skills-optional-recommended); a quick reference for the most common pairings:

| Stage / phase | Companion skills worth invoking |
|---|---|
| After Stage 0/1, before Stage 2 cleanup | `audit`, `security-audit`, `redesign-existing-projects` (if imported design feels generic-AI) |
| Visual polish (anywhere between Stage 1 and 12) | `polish`, `critique`, `accessibility-review`, `optimize`, `layout`, `typeset`, `clarify`, `colorize` / `bolder` / `delight` / `quieter` / `distill` (pick by direction) |
| Aesthetic direction (one per project) | `high-end-visual-design`, `minimalist-ui`, `industrial-brutalist-ui`, `impeccable`, `emil-design-eng`, `design-taste-frontend` |
| Copy + content (Stage 5 emails, Stage 8 privacy text, Stage 9/10 marketing pages) | `marketing:content-creation`, `marketing:email-sequence`, `marketing:brand-review`, `brand-voice:enforce-voice`, `brand-voice:generate-guidelines`, `design:ux-copy` |
| Pre-launch (Stage 12 companions) | `audit`, `security-audit`, `accessibility-review`, `optimize`, `legal:compliance-check` |
| Post-launch iteration | `superpowers:test-driven-development`, `superpowers:executing-plans`, `marketing:performance-report`, `marketing:campaign-plan` |

**My discipline**: I don't auto-invoke these. The user pulls them in when they want them, OR I suggest one when its specific value clearly matches the moment (e.g., "before we move to Stage 2, would you like me to run the `audit` skill against the imported source so we know what to prioritize during cleanup?"). If the user asks "what skills should I have for this?" I point them at the README mapping.

If a companion skill isn't installed in the user's environment, I report that cleanly ("the `audit` skill isn't loaded in your environment; if you want it, install via X — otherwise I'll run a manual audit") and proceed without it.

## Common command patterns the user might say

When the user says one of these, I run the corresponding flow:

| User says | I do |
|---|---|
| "Set up production for [site]" | All 13 stages, starting with onboarding |
| "Add analytics to [existing site]" | Stages 6 + 7 + 8 (skip earlier scaffolding) |
| "Set up the cookie banner" | Stage 8 alone, with quick prompt for posture |
| "Wire affiliate tracking" | Stage 10, after confirming the membership platform from Stage 9 |
| "Migrate [site] from [platform] to production" | Stage 0 with that platform pre-selected, then full flow |

## Outputs of a successful run

When all 13 stages complete, the user has:

- A Railway-deployed Vite/React/Express site at their chosen domain
- PostHog Live Events showing `$pageview` + custom events with deploy + variant super-properties on every event
- Clarity recordings with PostHog `distinct_id` + `session_id` as custom identifiers (cross-tool pivoting works)
- Resend sending welcome / autoresponder / notification emails from a verified domain
- A consent banner appearing in the configured locales, with version sentinels persisted in localStorage
- Optional: Whop subscription events forwarded server-side to PostHog with affiliate attribution joined from a DB lookup
- Optional: branded short URLs serving as stable indirection for affiliate URLs
- A set of saved PostHog dashboards with a runbook explaining how to read each card
- A repo-level `migration-punchlist.md` archiving every dependency I discovered + how I resolved it
- An `.env.example` enumerating every required env var so future redeploys don't hit the build-time silent-darkness trap

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial skill created. Captures production-stack patterns: Readdy/Webflow/Framer import → Railway → PostHog reverse proxy → variants → Clarity → Resend → consent → membership integration → affiliate tracking. Verified Railway has Remote MCP with OAuth, PostHog MCP supports OAuth, Clarity MCP is API-token-paste only, Resend has no MCP yet (REST API + first-key dashboard click). |
| 2026-05-08 | Major reframe to non-coder-first audience. Added Automation Contract at top stating that I (Claude) execute all technical work; the user provides decisions and a small number of inputs. Added user-expertise mode (`non_coder` / `developer` / `expert`) and onboarding mode (`all_upfront` / `phase_by_phase`) — both saved to skill config and consulted at every stage. Stripped all personal/brand-specific references; locked placeholder conventions (Maya's Consulting / mayasconsulting.com / mayas.link). Removed trademarked terminology. |
