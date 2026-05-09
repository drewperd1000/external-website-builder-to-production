# Stage 5: Email Provider

**Last verified: 2026-05-08.** Re-search `Resend MCP server <year>` before this stage if older than 60 days; an MCP didn't exist as of 2026-05 but may have shipped since.

> **🤖 Re-grounding (re-read at start of stage)**: I (Claude) wire up transactional email — install the SDK, write the transport abstraction, populate the form endpoints in `server.js`, write the email templates, set the Railway env vars, and run the end-to-end test. The user's actions: (1) sign up at the email provider's dashboard once and paste the API key into a project-local file, (2) add DNS records the provider gives me (or grant me a Cloudflare API token so I can do it via Cloudflare's API). Everything else I do autonomously.

## Required-values check (run at stage start)

I read `<project>/.skill-config.json` per the [universal pattern in SKILL.md](SKILL.md#required-values-check-pattern-universal--runs-at-start-of-every-stage). Stage 5 needs:
- `email.provider` — default to `"resend"` if unset; I confirm with the user.
- `email.sending_domain` — must be a domain the user controls. If unset, I prompt; this is blocking because DKIM/SPF setup needs it.
- `email.from_address` — full RFC-5322 form like `"Brand Name <hello@<sending-domain>>"`. If unset, I propose `"<project.brand> <hello@<sending-domain>>"` and confirm.
- `email.notify_to` — where contact-form / signup notifications go (typically the user's own inbox or a team alias). If unset, default to `hello@<sending-domain>` and confirm.
- The user's API key, pasted into `<project>/.secrets/<provider>-api-key.txt` — I prompt for this when I need it. **Parallel work I can do while waiting**: write the transport abstraction module, the email templates, and the form-endpoint wiring in `server.js`. The actual `railway variable set` of the key is the only step that blocks on the paste.

## What I'm doing in this stage

Wiring up transactional email so Stage 3's `/api/newsletter` and `/api/contact` endpoints (and any other forms from Stage 0) actually send mail. Building behind a swappable transport abstraction so future provider migrations are mechanical.

**Time**: 10–20 minutes of automated work, plus DNS-verification wait time (usually 5 minutes to a few hours, runs in parallel with other work).

## Default recommendation: Resend

Why I default to Resend:
- Strong deliverability + inbox placement
- Clean REST API, low conceptual surface
- Cheap free tier (3,000 emails/month, 100/day) — plenty for marketing signups
- Well-maintained TypeScript SDK

If the user (per onboarding) chose a different ESP, the structure of this stage stays the same — only Step 4 (the transport implementation) changes. Alternative implementations for AWS SES, Postmark, Mailgun, SendGrid are at the bottom of this file.

## Prerequisites

- Stage 4 complete (Railway service deployed).
- A sending domain the user controls (cannot send from `gmail.com`, `yahoo.com`, etc.).
- An email address on that domain that receives notifications (typically `hello@<user-domain>` or `support@<user-domain>`). If the user hasn't provisioned an inbox yet, I prompt them — Cloudflare Email Routing is the cheapest option (free, forwards to existing Gmail/etc.) for solo founders.

## My execution sequence

### Step 1: Resend account + initial API key

Resend has no MCP server as of 2026-05, and the first API key requires a dashboard click. I tell the user:

```
I need to set up your Resend account. This takes about 3 minutes:

  1. Please open https://resend.com/signup in your browser (I'll open it in a tab via Chrome MCP if it's connected; otherwise visit directly).
     just visit it directly).
  2. Sign up with your email + password (or use Google/GitHub OAuth).
  3. Verify your email when Resend sends the confirmation.
  4. Once you're in the dashboard, click "API Keys" → "Create API Key".
  5. Pick "Full Access" permission and click "Add".
  6. Resend shows the key ONCE. Copy it — it starts with "re_".
  7. Paste it here, and I'll save it to a project-local secrets file
     so it doesn't end up in chat history.
```

When the user pastes the key, I:
1. Create `<project>/.secrets/` directory if it doesn't exist
2. Add `.secrets/` to `.gitignore` if not already there
3. Write the key to `<project>/.secrets/resend-api-key.txt`
4. Confirm the file exists and is gitignored

After this initial bootstrap, Resend's REST API can mint additional sub-keys via `POST /api-keys` programmatically — I do that on the user's behalf if scoped keys per environment are needed later.

### Step 2: I verify the sending domain

This is the longest-pole step in Stage 5. DNS verification can take minutes to hours.

#### Path A: I do everything except DNS-record-paste (default for non_coder)

1. I call Resend's API to register the domain: `POST /domains` with the user's sending domain
2. Resend's response includes a `records` array — DKIM, SPF, optional MX + tracking CNAMEs
3. I tell the user the exact records to add at their DNS provider, with click paths:

```
Resend needs these DNS records added to mayasconsulting.com.

If your DNS is on Cloudflare:
  1. Open Cloudflare → Domains → mayasconsulting.com → DNS → Records
  2. Click "Add record" for each of the 4 records below
  3. (Easy mode: click "Bulk Edit" instead and paste this block)

The records:
  Type:  TXT
  Name:  resend._domainkey
  Value: p=MIGfMA0GCSqGSIb3DQEB... (long DKIM key Resend gave us)
  TTL:   Auto

  Type:  TXT
  Name:  send
  Value: v=spf1 include:amazonses.com ~all
  Note:  ⚠️ See "SPF merge" note below if you already have an SPF record.

  ... (and so on)

Tell me when you've added all of them. I'll click "Verify" in Resend.
```

4. Once the user confirms, I call Resend's `POST /domains/{id}/verify` endpoint
5. I poll status via `GET /domains/{id}` until status flips to `verified` (or timeout after 30 minutes — DNS can be slow)

#### Path B: Cloudflare API automation (faster if user grants me a token)

If the user has a Cloudflare API token (or grants me one for this project), I add the DNS records via Cloudflare's API automatically. Total time: 5 minutes for verification to settle, vs 30+ minutes for the human path.

I ask once during onboarding (or here if not yet asked):

```
Quick option: do you have a Cloudflare API token I can use to add these
DNS records automatically? (Saves you ~5 minutes of clicking.)

  (a) Yes — I'll paste it (will save to .secrets/cloudflare-api-token.txt)
  (b) No, I'll add the records manually
  (c) I don't have a Cloudflare API token; help me create one
```

If (a) or (c), I store the token in `.secrets/cloudflare-api-token.txt` and add records via Cloudflare's `POST /zones/{zone_id}/dns_records` endpoint, then verify via Resend's API.

#### SPF merge gotcha (I always check)

If the user already has an SPF record (common — Google Workspace, Microsoft 365, etc. add their own), they CANNOT have two TXT records starting with `v=spf1`. The records must be merged:

```
# Before (existing record from Google Workspace):
v=spf1 include:_spf.google.com ~all

# After (merged with Amazon SES that Resend uses):
v=spf1 include:_spf.google.com include:amazonses.com ~all
```

Before I tell the user to add the SPF record, I run `dig TXT <user-domain> +short` to check what's already there. If an existing `v=spf1` record exists, I produce the merged version instead of asking the user to add a duplicate.

#### Verification check

Once DNS propagates, I run `dig TXT resend._domainkey.<user-domain> +short` to confirm DKIM is visible. Then I call Resend's verify endpoint and confirm the status flips to `"verified"`.

### Step 3: I install the Resend SDK

```
npm install resend
```

The SDK is ESM + has TypeScript types. No additional config needed.

### Step 4: I implement the transport abstraction

I create `server/email.js` with a transport interface. This is the swappable boundary — future SES / Postmark / Mailgun migrations only edit this file.

```js
/**
 * Email service — generic transport interface with Resend default.
 * To swap providers: implement EmailTransport, replace getTransport().
 * The rest of the codebase calls sendEmail() and doesn't know about the
 * underlying provider.
 */
import { Resend } from "resend";

// Fail-fast on missing env vars in production. The "example.com" fallbacks
// that AI-generated boilerplate often ships are silently broken — Resend
// rejects them with 422, but the bundle deploys, the server boots, and
// the first form submit fails in the visitor's face. Throwing at import
// time means the deploy fails LOUDLY, before any visitor is affected.
function requireEnv(name) {
  const v = process.env[name];
  if (v === undefined || v === null || v === "") {
    if (process.env.NODE_ENV === "production") {
      throw new Error(
        `[email] Required env var ${name} is not set. ` +
        `Set it on Railway: railway variable set ${name}="<value>"`
      );
    }
    // In development we tolerate missing vars so dev servers boot —
    // the console transport will be picked up below and emails just log.
    return null;
  }
  return v;
}

export const EMAIL_FROM = requireEnv("EMAIL_FROM");
export const EMAIL_NOTIFY_TO = requireEnv("EMAIL_NOTIFY_TO");
export const SITE_URL = requireEnv("SITE_URL");

// ─── Resend transport ─────────────────────────────────────────────
function createResendTransport() {
  if (!process.env.RESEND_API_KEY) {
    throw new Error("RESEND_API_KEY not set");
  }
  const resend = new Resend(process.env.RESEND_API_KEY);
  return {
    async send(msg) {
      try {
        const { data, error } = await resend.emails.send({
          from: msg.from || EMAIL_FROM,
          to: msg.to,
          replyTo: msg.replyTo,
          subject: msg.subject,
          html: msg.html,
          text: msg.text,
        });
        if (error) {
          console.error("[email/resend] send error:", error);
          return { success: false, error };
        }
        return { success: true, messageId: data?.id };
      } catch (err) {
        console.error("[email/resend] exception:", err);
        return { success: false, error: err };
      }
    },
  };
}

// ─── Console transport (development fallback) ─────────────────────
const consoleTransport = {
  async send(msg) {
    console.log(
      `\n📧 [Email DEV] ─────────────────────────────────\n` +
      `  To:      ${msg.to}\n` +
      `  Subject: ${msg.subject}\n` +
      `  Body:    ${msg.text ?? "(HTML only)"}\n` +
      `─────────────────────────────────────────────────\n`,
    );
    return { success: true, messageId: `dev-${Date.now()}` };
  },
};

// ─── Transport selection ──────────────────────────────────────────
let _transport = null;
function getTransport() {
  if (_transport) return _transport;
  if (process.env.RESEND_API_KEY) {
    _transport = createResendTransport();
  } else {
    if (process.env.NODE_ENV === "production") {
      console.warn("[email] RESEND_API_KEY unset in production — emails will log to console only");
    }
    _transport = consoleTransport;
  }
  return _transport;
}

// ─── Public API ───────────────────────────────────────────────────
export async function sendEmail(msg) {
  const t = getTransport();
  return t.send(msg);
}
```

**Key design points**:
- **Fail-fast on missing addresses, lazy on the transport.** Required addresses (`EMAIL_FROM`, `EMAIL_NOTIFY_TO`, `SITE_URL`) throw at import time in production, so the deploy fails loudly if the user forgot to set them on Railway. The Resend transport itself is lazy: if `RESEND_API_KEY` is unset, the module falls through to the console transport and the server boots — appropriate for staging or dev environments where transactional email isn't load-bearing. The reason for the asymmetry: the addresses ship as build-time invariants the user verified at Stage 1, so missing-at-deploy means broken-everywhere; the API key is more dynamic and might genuinely be unset on a non-prod environment.
- **`EMAIL_FROM` must be on a Resend-verified domain.** Stage 5's verification step confirms this. If the user later changes the sending domain, the deploy will silently send 422s — gotcha section below covers this.

### Step 5: I wire form endpoints in `server.js`

I update Stage 3's `server.js` to populate the form-handler placeholders:

```js
import { sendEmail, EMAIL_FROM, EMAIL_NOTIFY_TO, SITE_URL } from "./server/email.js";

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

app.post("/api/newsletter", async (req, res) => {
  const email = (req.body?.email || "").toString().trim().toLowerCase();
  if (!email || !EMAIL_RE.test(email) || email.length > 254) {
    return res.status(400).json({ ok: false, error: "Invalid email address." });
  }
  try {
    await Promise.all([
      sendEmail({
        to: email,
        subject: "Welcome",
        text: newsletterWelcomeText(),
        html: newsletterWelcomeHtml(),
      }),
      sendEmail({
        to: EMAIL_NOTIFY_TO,
        subject: `New newsletter signup: ${email}`,
        text: `${email} subscribed at ${new Date().toISOString()}`,
      }),
    ]);
    return res.json({ ok: true });
  } catch (err) {
    console.error("[/api/newsletter] failed:", err);
    return res.status(500).json({ ok: false, error: "Send failed." });
  }
});

app.post("/api/contact", async (req, res) => {
  const { full_name, email, subject, message, inquiry_type, organization } = req.body || {};
  if (
    !full_name || !email || !subject || !message ||
    !EMAIL_RE.test(email) ||
    full_name.length > 200 || email.length > 254 ||
    subject.length > 300 || message.length > 5000
  ) {
    return res.status(400).json({ ok: false, error: "Invalid submission." });
  }
  try {
    await Promise.all([
      sendEmail({
        to: email,
        subject: "We got your message",
        text: contactAutoresponderText(full_name),
        html: contactAutoresponderHtml(full_name),
      }),
      sendEmail({
        to: EMAIL_NOTIFY_TO,
        replyTo: email,
        subject: `[Contact] ${subject}`,
        text: contactNotificationText({ full_name, email, subject, message, inquiry_type, organization }),
      }),
    ]);
    return res.json({ ok: true });
  } catch (err) {
    console.error("[/api/contact] failed:", err);
    return res.status(500).json({ ok: false, error: "Send failed." });
  }
});
```

I add similar handlers for any other forms Stage 0 catalogued (support, beta-signup, lead-magnet, etc.). The structure stays the same: validate → send confirmation to user + notification to team → return success.

### Step 6: I write email templates

I keep templates simple — heavy styling hurts deliverability (spam filters score against image-heavy / CSS-heavy mail). Inline styles only, no `<style>` blocks.

Example shell + welcome template. **Color values shown are generic neutrals** — at write time I extract brand colors from the user's existing site (CSS variables, Tailwind config, brand kit if provided) and substitute. If no brand colors are detectable, I keep the neutrals below and ask the user to provide brand hex codes.

```js
// I substitute brand colors at write time. Variables I look for:
const BRAND_BG = "<brand-background>";        // e.g., "#ffffff" or off-white
const BRAND_ACCENT = "<brand-accent>";        // e.g., logo color
const BRAND_TEXT_PRIMARY = "<text-primary>";  // typically near-black
const BRAND_TEXT_MUTED = "<text-muted>";      // typically gray
const BRAND_BORDER = "<border-color>";        // light gray for dividers

function emailShell(bodyHtml) {
  return `<!doctype html>
<html><body style="margin:0;padding:0;background:${BRAND_BG};font-family:-apple-system,BlinkMacSystemFont,'Segoe UI',Roboto,sans-serif;color:${BRAND_TEXT_PRIMARY};">
  <table role="presentation" cellpadding="0" cellspacing="0" border="0" width="100%" style="background:${BRAND_BG};padding:40px 20px;">
    <tr><td align="center">
      <table role="presentation" cellpadding="0" cellspacing="0" border="0" width="560" style="max-width:560px;background:#ffffff;border-radius:12px;padding:40px;text-align:left;">
        <tr><td style="padding-bottom:28px;border-bottom:1px solid ${BRAND_BORDER};">
          <div style="font-size:14px;letter-spacing:2px;text-transform:uppercase;color:${BRAND_ACCENT};font-weight:600;">Brand Name</div>
        </td></tr>
        <tr><td style="padding-top:28px;font-size:15px;line-height:1.6;color:${BRAND_TEXT_PRIMARY};">${bodyHtml}</td></tr>
        <tr><td style="padding-top:32px;border-top:1px solid ${BRAND_BORDER};font-size:12px;color:${BRAND_TEXT_MUTED};line-height:1.5;">
          You're receiving this because you signed up at <a href="${SITE_URL}" style="color:${BRAND_ACCENT};text-decoration:none;">${SITE_URL.replace(/^https?:\/\//, '')}</a>.
        </td></tr>
      </table>
    </td></tr>
  </table>
</body></html>`;
}

function newsletterWelcomeHtml() {
  return emailShell(`
    <h1 style="font-size:24px;font-weight:300;margin:0 0 16px 0;">Welcome.</h1>
    <p style="margin:0 0 16px 0;">You're in. Thanks for joining the list.</p>
    <p style="margin:0 0 16px 0;">A few times a month you'll hear from us with insights and early-access updates. Unsubscribing is one click whenever you've had enough.</p>
  `);
}

function newsletterWelcomeText() {
  return `Welcome.

You're in. Thanks for joining the list.

A few times a month you'll hear from us with insights and early-access updates. Unsubscribing is one click whenever you've had enough.

—
You're receiving this because you signed up at ${SITE_URL}.`;
}
```

I always write both HTML and text variants. Some email clients render text only; some flag HTML-only mail as suspicious.

**For user-provided strings** (full name, message body) interpolated into HTML, I use an `escapeHtml` helper to prevent injection:

```js
function escapeHtml(s) {
  return String(s).replace(/[&<>"']/g, (c) => ({
    "&": "&amp;", "<": "&lt;", ">": "&gt;", '"': "&quot;", "'": "&#39;",
  }[c]));
}
```

Used inside template functions: `<h1>Got it, ${escapeHtml(firstName)}.</h1>`.

For the actual brand voice / copy in each template, I draft based on the user's brand description from onboarding, then ask them to review before going live.

### Step 7: I set Resend env vars on Railway

I run (from the canonical pattern in Stage 4 Step 3):

```
railway variable set RESEND_API_KEY="$(cat .secrets/resend-api-key.txt)"
railway variable set EMAIL_FROM="Brand Name <hello@<user-domain>>"
railway variable set EMAIL_NOTIFY_TO="hello@<user-domain>"
```

I substitute the real brand name and domain at write time. **`RESEND_API_KEY` is server-side only** — I never prefix with `VITE_` (would bake into the client bundle, exposing the key to anyone viewing source).

Then I trigger a redeploy: `railway up`.

### Step 8: I run end-to-end test

I submit a test newsletter signup via curl to the deployed site:

```
curl -X POST https://<user-domain>/api/newsletter \
  -H "Content-Type: application/json" \
  -d '{"email":"<test-email>"}'
```

Expected response: `{"ok":true}`.

Then I verify:
- The user's test email address received the welcome email (I ask the user to confirm this — it's the only thing I can't autonomously verify since I can't read their inbox)
- The notification email landed at `EMAIL_NOTIFY_TO` (same — user confirms)
- Resend dashboard shows both sends with status "delivered" — I check this via Resend's API: `GET /emails` filtered by recent timestamp

If sends fail:
- I check `railway logs` for `[email/resend] send error:` entries (current CLI streams by default)
- I query Resend's API for the failed message and report the error code
- Common causes: domain not yet verified, `from` address not on a verified domain, key typo

### Step 9: I update privacy policy

Once Resend is in the data path, I add it to the privacy policy as a sub-processor. Stage 8 owns the privacy policy file; I add a draft entry now that Stage 8 picks up:

```markdown
## Sub-processors
- Resend (email delivery) — receives form submissions for newsletter and
  contact processing. Stores email metadata for 30 days. Privacy policy:
  https://resend.com/legal/privacy-policy
```

For users with a DPA (Data Processing Agreement) requirement (GDPR scope), I note that Resend supports DPAs — request via their support if processing EU data at scale.

## Verification (autonomous)

I run all of these myself before declaring Stage 5 complete:

| Check | What I do | Pass criteria |
|---|---|---|
| Domain verified | `GET /domains/{id}` via Resend API | status: "verified" |
| Railway env vars set | `railway variable list` filtered on `RESEND_*`, `EMAIL_*` | All three present |
| Form endpoints respond | `curl POST /api/newsletter` and `/api/contact` against deployed URL | HTTP 200, `{ok:true}` |
| Resend logs sends | `GET /emails` via Resend API (filter recent) | Both test sends visible with status "delivered" |
| Privacy policy mentions Resend | Read draft policy file | Resend listed as sub-processor |

I report results to the user:

```
✅ Stage 5 complete:
   • Resend account active, sending domain mayasconsulting.com verified
   • Transport abstraction at server/email.js (Resend default; swappable)
   • /api/newsletter and /api/contact wired in server.js
   • Welcome + autoresponder + notification templates created
   • Test sends: 2 delivered (1 welcome to your test email, 1 notification
     to hello@mayasconsulting.com)
   • Privacy policy draft updated with Resend as sub-processor

Ready for Stage 6 (PostHog)?
```

## Alternative ESP implementations

The transport abstraction (Step 4) is the swap point. Each alternative ESP implements the same `{ send: async (msg) => { success, messageId | error } }` contract.

**AWS SES** (`@aws-sdk/client-sesv2`):

```js
import { SESv2Client, SendEmailCommand } from "@aws-sdk/client-sesv2";
const ses = new SESv2Client({ region: process.env.AWS_REGION });
return {
  async send(msg) {
    const cmd = new SendEmailCommand({
      FromEmailAddress: msg.from || EMAIL_FROM,
      Destination: { ToAddresses: [msg.to] },
      Content: { Simple: {
        Subject: { Data: msg.subject },
        Body: { Text: { Data: msg.text || "" }, Html: { Data: msg.html || "" } },
      }},
    });
    try {
      const out = await ses.send(cmd);
      return { success: true, messageId: out.MessageId };
    } catch (err) {
      return { success: false, error: err };
    }
  },
};
```

**Postmark** (`postmark`):

```js
import postmark from "postmark";
const client = new postmark.ServerClient(process.env.POSTMARK_SERVER_TOKEN);
return {
  async send(msg) {
    try {
      const out = await client.sendEmail({
        From: msg.from || EMAIL_FROM,
        To: msg.to,
        Subject: msg.subject,
        TextBody: msg.text,
        HtmlBody: msg.html,
      });
      return { success: true, messageId: out.MessageID };
    } catch (err) {
      return { success: false, error: err };
    }
  },
};
```

**Mailgun, SendGrid, Sendinblue**: same pattern. Each SDK has its own `send` shape; I adapt the `{ success, messageId | error }` return contract.

## Common gotchas I handle automatically

- **`from` address must be on a verified domain**: sending `From: hello@unverified.com` returns 422 from Resend. I always check `EMAIL_FROM` matches a verified domain before saving.
- **Test email lands in spam**: usually a DKIM/SPF DNS misconfiguration. I run `dig TXT <user-domain> +short` to verify both records present + valid. Resend dashboard's domain page also lints this.
- **Hitting daily quota in development**: Resend free tier is 100/day. If a user runs `npm run dev` repeatedly with form submits, they hit the cap fast. I default the dev environment to use the console transport (don't set `RESEND_API_KEY` locally).
- **Replying to autoresponder bounces back to user**: I set `replyTo: EMAIL_NOTIFY_TO` on the autoresponder so user replies go to the team inbox, not a no-reply address.
- **Email mailbox provisioning lag**: I provision the inbox and verify it's receiving BEFORE deploying the live form endpoints. Otherwise users sign up and the user never sees the notification.

## Self-research instruction

Before running this stage, I web-search:
- `Resend MCP server <year>` — none as of 2026-05; if one exists now, I use it instead of REST
- `Resend API rate limits <year>` — quotas may have shifted
- `<chosen-ESP> Cloudflare DNS auto-provision <year>` — if a DNS-auto-add integration shipped, Step 2 Path A becomes one-click

## Outputs

After Stage 5:
- Verified sending domain in Resend (or alternative ESP)
- `server/email.js` with swappable transport
- Form endpoints wired in `server.js`
- Email templates with brand-appropriate copy + styling
- Real emails flowing from real form submits
- Privacy policy draft updated to list ESP as sub-processor

## Changelog

| Date | Change |
|---|---|
| 2026-05-08 | Initial. No Resend MCP exists in 2026-05; first API key requires dashboard click. Sub-keys + domains programmatic via REST. DNS records always require user to add to DNS provider (or Cloudflare API side-channel automation). |
| 2026-05-08 | Reframed for non-coder audience: every "Run X" / "Edit Y" reframed to first-person Claude voice for actions Claude actually executes. Step 1 user instructions made honest: signup is a user action ("Please open https://resend.com/signup; I open it via Chrome MCP if connected") rather than a fictional Claude-drives-browser claim. Step 2 split into Path A (user adds DNS records, fastest for one-domain) vs Path B (Cloudflare API automation if user grants me a token, fastest if they have one). Replaced literal author-domain examples with `<user-domain>` substitution markers. Stripped author-specific knowledge-base references and replaced author-home-directory secret paths with project-local equivalents. |
