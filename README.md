<div align="center" style="background-color:#0a0a0a; padding: 44px 20px 40px; border-radius: 12px;">
  <h1 style="color:#f0ede8; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif; font-weight:800; font-size:3em; letter-spacing:-0.03em; margin:0;">
    Send<span style="color:#f5a623;">Fleet</span>
  </h1>
  <p style="color:#8a8480; font-size:0.9em; margin-top:8px;">
    Your SES or ours. One API. Zero friction.
  </p>
</div>

<br>

> **SendFleet is a transactional email API with two sending modes.**
> Route mail through your own AWS SES account (BYOC) or through SendFleet's managed sending
> infrastructure. Same endpoint, same JSON payload, same response shape. On BYOC, recipients,
> subjects, and bodies are never written to our database. No SDK required.

<br>

## Contents

- [Two modes. One endpoint.](#two-modes-one-endpoint)
- [Quickstart](#quickstart)
- [Send endpoint](#send-endpoint)
- [Attachments](#attachments)
- [Correlating a send with its events](#correlating-a-send-with-its-events)
- [Account endpoint](#account-endpoint)
- [API keys](#api-keys)
- [BYOC: Bring Your Own SES](#byoc-bring-your-own-ses)
- [Shared SES: Managed infrastructure](#shared-ses-managed-infrastructure)
- [Plans and pricing](#plans-and-pricing)
- [DNS and deliverability](#dns-and-deliverability)
- [Webhooks](#webhooks)
- [Error codes](#error-codes)
- [Bounces, complaints, and suspension](#bounces-complaints-and-suspension)
- [Dashboard](#dashboard)
- [Billing behaviour](#billing-behaviour)
- [Security](#security)
- [Limits at a glance](#limits-at-a-glance)
- [Get started](#get-started)

<br>

## Two modes. One endpoint.

SendFleet operates two distinct sending paths. You choose when you add a sender domain — and the
API call looks identical either way.

The routing decision is automatic. It is made from the **sending mode attached to the domain** in
your `from_email`. That mode is fixed when the domain is added and cannot be changed afterwards:
a BYOC domain is registered as an identity in **your** AWS SES account, a Shared SES domain is
registered in **SendFleet's**.

> **Before you switch plans, read this.** A domain's sending mode is permanent. If you move from a
> BYOC plan to a Managed plan — or the other way — your existing domains keep their original mode
> and your sends start failing with `403 domain_mode_mismatch`. Add fresh domains for the new mode
> and retire the old ones. Switching is fully self-serve and takes minutes, not a support ticket.

| | BYOC | Shared SES |
|---|---|---|
| **Sends through** | Your own AWS SES account | SendFleet's managed SES |
| **Recipients, subjects, bodies stored on our servers** | Never | Yes — 14 days, in your dashboard |
| **Sender reputation** | Isolated to your AWS account | Shared pool across Managed accounts |
| **AWS account required** | Yes (yours, out of sandbox) | No |
| **Manual approval required** | No | No |
| **Monthly send limit** | 50 (Free) or unlimited ($9) | 5,000 / 25,000 / 100,000 |
| **Email log dashboard** | — (use your SES / CloudWatch) | Full 14-day history |
| **Delivery outcomes** | In your own AWS account | Managed for you |
| **Webhook delivery events** | — | ✓ |
| **Attachments** | ✓ | ✓ |

<br>

## Quickstart

**You need:** a SendFleet account, a verified sender domain, and an API key. That is the whole
setup. No SDK, no agent, no build step.

```bash
curl -X POST https://api.sendfleet.net/v1/send \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "to": "user@example.com",
    "from_email": "hello@yourdomain.com",
    "from_name": "Your App",
    "subject": "Welcome",
    "message": "Thanks for signing up."
  }'
```

```json
{
  "success": true,
  "message": "Email queued for delivery via your SES account.",
  "message_id": "1a2b3c4d-5e6f-7890-abcd-ef1234567890",
  "mode": "byoc"
}
```

A `200 OK` means the message was **accepted and queued**. Delivery is asynchronous, so this is not
yet a delivery confirmation. Use [webhooks](#webhooks) to learn what happened.

The same call in other languages — the request body is identical:

```python
import requests

requests.post(
    "https://api.sendfleet.net/v1/send",
    headers={"X-API-Key": "YOUR_API_KEY"},
    json={
        "to": "user@example.com",
        "from_email": "hello@yourdomain.com",
        "from_name": "Your App",
        "subject": "Welcome",
        "message": "Thanks for signing up!",
    },
    timeout=10,
)
```

```javascript
await fetch("https://api.sendfleet.net/v1/send", {
  method: "POST",
  headers: { "X-API-Key": "YOUR_API_KEY", "Content-Type": "application/json" },
  body: JSON.stringify({
    to: "user@example.com",
    from_email: "hello@yourdomain.com",
    from_name: "Your App",
    subject: "Welcome",
    message: "Thanks for signing up!",
  }),
});
```

<br>

## Send endpoint

**`POST https://api.sendfleet.net/v1/send`**

Every request sends `Content-Type: application/json` and your key in the `X-API-Key` header.
Request bodies are JSON; responses are always JSON. This is the only sending endpoint.

### Request fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `to` | email or email[] | **Yes** | One address, or a list. `to`, `cc` and `bcc` together hold at most **50** recipients (the Amazon SES per-message limit), and no address may appear twice across them. |
| `cc` | email or email[] | No | Copy recipients, shown to everyone in the `Cc` header. |
| `bcc` | email or email[] | No | Blind-copy recipients. They receive the message and appear in no header. |
| `from_email` | email | Effectively yes | Must belong to a verified, send-ready domain in your account. **This field drives routing.** Omit it and it falls back to your account login email, which is almost never a sending domain, so the call fails with `domain_not_registered`. |
| `from_name` | string | No | Display name, rendered as `From: Your App <hello@yourdomain.com>`. Max **156** characters. |
| `subject` | string | No | Max **256** characters. Falls back to `(no subject)`. Spam filters penalise subject-less mail — always set one. |
| `message` | string | No | Plain-text body. Strongly recommended even when you also send `html`. |
| `html` | string | No | HTML body. Clients that render HTML show this; the rest fall back to `message`. Use inline CSS only. |
| `reply_to` | email | No | Where replies should go. Defaults to `from_email`. |
| `attachments` | object[] | No | Up to **20** per message. See [Attachments](#attachments). |

**At least one of `message`, `html`, or `attachments` must be present.** None is individually
required, but a request carrying none of the three is rejected with a `400`.

### Several recipients

One request sends **one message** to everyone in `to`, `cc` and `bcc`:

```json
{
  "to": ["alice@example.com", "bob@example.com"],
  "cc": "accounts@example.com",
  "bcc": ["archive@yourdomain.com"],
  "from_email": "orders@yourdomain.com",
  "subject": "Your order #1042",
  "message": "Thanks for your order!"
}
```

- **Every recipient counts as one send**, the way Amazon SES counts them — the request above uses
  four of your monthly allowance. A send that needs more than your plan has left is refused whole
  with `429 usage_limit_exceeded`; it never goes to only some of its recipients.
- **At most 50 recipients** across the three fields, and **no address twice** (compared
  case-insensitively). Either is a `400`.
- **On Shared SES every recipient is its own entry** in your email log and gets its own webhooks,
  all under the send's one `idempotency_key`, with the address in `recipient`.
- **The Free plan's Shared smoke test allows no `cc` or `bcc`** — it reaches your own address only.
  Free BYOC sending supports both.
- **A SendFleet test address must be a send's only recipient.**

### Successful response — `200 OK`

```json
{
  "success": true,
  "message": "Email queued for delivery.",
  "message_id": "9f8e7d6c-5b4a-3210-fedc-ba9876543210",
  "idempotency_key": "8f14e45fceea167a5a36dedd4bea2543",
  "mode": "shared",
  "attachments": 2,
  "warnings": [
    "DMARC record not found. Add 'v=DMARC1; p=none;' as a TXT record on _dmarc.yourdomain.com to improve inbox placement."
  ]
}
```

| Field | Mode | Description |
|---|---|---|
| `success` | both | Always `true` on `200`. |
| `message` | both | Human-readable confirmation. Wording differs between modes — **do not branch on it**. |
| `message_id` | both | Opaque reference for this accepted request. Quote it to support about a specific send. It does **not** appear on any webhook. |
| `idempotency_key` | **Shared SES only** | The correlation key for this send. The same value appears on every webhook fired for it. Absent on BYOC responses. |
| `mode` | both | `"byoc"` or `"shared"` — which path delivered the message. |
| `attachments` | both | Count of attachments accepted. Present only when the request included attachments. |
| `recipients` | both | Number of recipients across `to`, `cc` and `bcc`, each counted as one send. Present only when there is more than one. |
| `warnings` | **Shared SES only** | Non-blocking deliverability gaps, such as a missing MAIL FROM or DMARC record. |

**Warnings never block delivery.** The email is still sent. They flag DNS gaps that measurably hurt
inbox placement — resolve them from **Dashboard → Domains**.

### HTML email that actually lands

| Rule | Why |
|---|---|
| Use inline CSS only | `<style>` blocks are stripped by Gmail and most mobile clients. |
| Use table-based layouts | Flexbox and Grid are ignored by Outlook, which renders with the Word engine. |
| Host images on absolute HTTPS URLs | Remote images are blocked by default in most clients. |
| Always include a plain-text body | It is the fallback for every client that cannot render HTML. |
| Keep the HTML under 100 KB | Gmail clips messages past roughly 102 KB. |

<br>

## Attachments

Attachments work on **every plan, in both modes.** They are held only until the message is
delivered, then discarded.

```json
{
  "to": "user@example.com",
  "from_email": "billing@yourdomain.com",
  "subject": "Your invoice",
  "message": "Your invoice is attached.",
  "attachments": [
    {
      "filename": "invoice.pdf",
      "content": "JVBERi0xLjQK...",
      "content_type": "application/pdf"
    }
  ]
}
```

| Attachment field | Type | Required | Notes |
|---|---|---|---|
| `filename` | string | **Yes** | Original filename including its extension. Max **255** characters. |
| `content` | string (base64) | **Yes** | File contents as a base64 string. Read the file as binary and encode it — do not pass raw bytes or a `data:` URI. |
| `content_type` | string (MIME) | No | e.g. `application/pdf`. Inferred from the filename when omitted. Must be an allowed type. |

| Limit | Value |
|---|---|
| Files per message | 20 |
| Per file | 25 MB |
| Total per message | 37 MB |
| Total request body | 58 MB |

Executable and script file types are rejected with a `400` — `.exe`, `.bat`, `.cmd`, `.msi`, `.js`,
`.jar`, `.dll`, `.scr`, `.vbs`, `.ps1` among others — **regardless of the `content_type` you
declare**. Past that, `content_type` must be on the allowed list, which covers the common document
(PDF, Word, Excel, OpenDocument, RTF), image, archive, calendar/contact, and audio/video formats.
Anything outside it is rejected, so check unusual file types before you depend on them.

Because base64 inflates the payload by roughly a third, a maximum-size attachment set lands around
49 MB on the wire — within the 58 MB body ceiling, but it is worth keeping an eye on if you also
send a large HTML body.

<br>

## Correlating a send with its events

Every accepted send is assigned a unique key, and on the managed path that key follows the message
all the way to delivery.

- **On Shared SES**, it is returned in the response as `idempotency_key` and appears on every
  webhook fired for that send. **Store it** — it is how you tie a response to its delivery events.
  A send to several recipients has one key and fires each event once per recipient, so match on
  `idempotency_key` and `recipient` together.
- **On BYOC** there is no webhook and no log, so there is nothing to correlate against. Keep the
  `message_id` for your own records and to quote to support.

`message_id` is the only reference on BYOC responses. It never appears on a webhook, so for webhook
correlation `idempotency_key` is the only field to use.

> **Retry safely.** These keys are correlation handles, not de-duplication tokens — SendFleet does
> not suppress a second identical call. If a send times out or returns `502`, check your own
> delivery state before retrying, or your recipient may get two copies. Webhooks are the reliable
> way to know whether the first attempt landed.

<br>

## Account endpoint

**`GET https://api.sendfleet.net/v1/me`**

Returns the authenticated account's identity, verification status, current plan, BYOC
configuration status, verified sending domains, and current-month usage. Same `X-API-Key` header.
The response is scoped to the account the key belongs to and never includes secrets.

```bash
curl https://api.sendfleet.net/v1/me -H "X-API-Key: YOUR_API_KEY"
```

```json
{
  "name": "Your Name",
  "email": "you@example.com",
  "is_verified": true,
  "current_plan": "growth",
  "byoc_usage_current_month": 3,
  "shared_usage_current_month": 742,
  "shared_delivered_current_month": 730,
  "shared_bounced_current_month": 4,
  "shared_complained_current_month": 0,
  "shared_rejected_current_month": 1,
  "byoc_configured": false,
  "verified_byoc_domains": [],
  "verified_shared_domains": ["mail.example.com"]
}
```

| Field | Description |
|---|---|
| `name`, `email` | Account identity. |
| `is_verified` | Sending is blocked until this is `true`. |
| `current_plan` | The plan whose allowance applies **right now**: `starter` (free), `byoc`, `managed_starter`, `growth`, or `pro`. An inactive, cancelled or `past_due` subscription reports `starter`, because that is what you are metered against — read this field, not your billing history, when you want to know your limit. |
| `byoc_usage_current_month` | BYOC sends this calendar month (UTC). **On Free BYOC this is the plan's one counter** and includes Shared smoke-test sends (see [Free plan: the Shared smoke test](#free-plan-the-shared-smoke-test)); it is the number checked against the 50. |
| `shared_usage_current_month` | Shared sends this calendar month. On a Managed plan this is exactly the number checked against your plan limit. On Free BYOC it only breaks out how many of your sends were smoke tests. |
| `shared_delivered_current_month` | Feeds your delivery rate. |
| `shared_bounced_current_month` | Feeds your bounce rate. |
| `shared_complained_current_month` | Feeds your complaint rate. |
| `shared_rejected_current_month` | Permanent rejections. |
| `byoc_configured` | Whether your IAM role has been validated. |
| `verified_byoc_domains` / `verified_shared_domains` | Verified domains per mode. |

A month with no activity returns zero rather than omitting the field. Previous months are not
included. `GET /v1/me` counts against the same 60 requests/minute limit as sending — call it at
startup or on a schedule, not before every individual email.

`GET https://api.sendfleet.net/health` is an unauthenticated liveness endpoint.

> **Base URL.** The API is served only from `https://api.sendfleet.net`. An integration still
> calling `https://sendfleet.net/api/send/` or `/api/me/` must switch to `/v1/send` and `/v1/me` on
> the API host; the headers, request body and responses are identical. The dashboard stays at
> `sendfleet.net/dashboard`.

<br>

## API keys

Create keys in **Dashboard → API Keys**. There is no limit — every plan includes unlimited keys.

| Property | Behaviour |
|---|---|
| **Format** | `<8-char-prefix>.<32-char-secret>`, dot-separated. Send the whole string — the prefix alone will not authenticate. |
| **Storage** | Only a SHA-512 hash is stored. A database breach cannot expose your raw secrets. |
| **Reveal** | Shown exactly once, immediately after creation. A lost key cannot be recovered. |
| **Header** | `X-API-Key`. Header names are case-insensitive, so `x-api-key` is fine. `Authorization` is not. |
| **Expiry** | Optional, set at creation. An expiry date cannot be extended afterwards. Expired keys are rejected and stay visible in your dashboard as an audit trail. |
| **Revoke** | Immediate and irreversible — the key stops working on the next request. Plan for up to two minutes if you need to be conservative. The record is kept so you retain the history of what existed and when it was disabled. |
| **Delete** | Permanent. The record, its hash, and its domain restrictions are removed and cannot be restored by anyone, including support. |
| **Domain scope** | Optional. A scoped key may only send from the selected domains; anything else returns `403` with `domain_not_allowed`. |
| **Rate limit** | **60 requests per minute per key**, on every plan. |

> **One scoped-key gotcha.** Deleting a domain removes it from every key's restriction list. If
> that empties a key's list, the key **silently regains access to all domains**. Check which keys
> depend on a domain before you remove it.

Use one key per environment, store keys in environment variables, and never put a key in
client-side code. Anything shipped to a browser or mobile app is readable by anyone who looks. If a
key is exposed, revoke it from the dashboard immediately.

After **10 failed authentication attempts from the same IP address within 60 seconds**, further
attempts from that address are temporarily blocked.

<br>

## BYOC: Bring Your Own SES

BYOC delivers your email through **your own AWS SES account**. Recipients, subjects, and bodies never
land in SendFleet's database, and your sender reputation is entirely your own.

**What you need:**
- An AWS account with SES approved for **production access** (out of sandbox). Sandbox accounts can
  only send to verified addresses and are capped at 200 emails a day, which is not usable for real
  sending.
- A domain you control, so you can add DNS records.
- Permission to create IAM roles in that AWS account.

**Setup in three steps:**

1. **Create the IAM role.** In your AWS account, create a role named exactly
   `SendFleet-BYOC-Role` — case-sensitive, and we only assume roles with that name. Paste in the
   trust policy from your dashboard: it already contains SendFleet's account ID and **your** unique
   ExternalId. Attach the permissions policy shown alongside it, which grants the SES actions
   needed to send and to manage your sending identities.
2. **Enter the Role ARN and region.** In **Dashboard → Settings → BYOC**, paste
   `arn:aws:iam::123456789012:role/SendFleet-BYOC-Role` and pick your SES region. We confirm the
   role can be assumed, that your SES account is out of sandbox, and that it has the required
   permissions. IAM changes can take a short while to propagate — if validation fails right after
   you create the role, wait a moment and retry.
3. **Add a sending domain.** We register the SES identity in **your** account and show you the DNS
   records to add.

> **Validate your role before upgrading to the $9 BYOC plan.** Purchasing BYOC requires a validated
> role ARN — you cannot check out the paid plan first and sort out IAM afterwards. Free BYOC has no
> such requirement, so it is the easiest way to prove the integration works before you pay.

### The ExternalId

Every BYOC configuration gets a unique, randomly generated **ExternalId** — 48 hex characters, 192
bits of entropy. It is generated when you first save your config, and it never changes.

It must appear in the `Condition` block of your IAM trust policy. This prevents the
[confused deputy attack](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html):
without it, anyone who learned your Role ARN could ask us to assume it on their behalf. Only
requests carrying the correct value can assume your role.

**Never drop the ExternalId condition.** A role that trusts our account without it is exploitable,
and there is no reason to remove it — the value is generated for you and pre-filled in the template
in your dashboard.

### How each send reaches your account

SendFleet assumes the role you created, in your account, for a short-lived session of **up to 15
minutes**, refreshed continuously, and calls SES with those temporary credentials. No long-lived
credential *of yours* is ever stored. The role grants only SES permissions, and you can revoke it at
any moment from your own AWS console — sending stops immediately.

### What we store

Recipients, subjects, and bodies for BYOC sends exist only in transit and in your own SES account.
SendFleet retains **no** of them — **only a monthly send counter**, used to enforce your plan limit.
(Attachments are the one exception: they are staged for the duration of delivery and then
discarded, exactly as they are on Managed plans. See [Attachments](#attachments).)

### Regions

`us-east-1`, `us-west-2`, `eu-west-1`, `eu-central-1`, `ap-southeast-1`, `ap-southeast-2`,
`ap-northeast-1`, `ap-south-1`, `ca-central-1`, `sa-east-1`.

### BYOC limits

- **Free BYOC** — 50 emails per month.
- **BYOC ($9/mo)** — no SendFleet cap. Your own AWS SES sending quota still applies.

The Free BYOC allowance also covers the [Shared smoke test](#free-plan-the-shared-smoke-test): one
monthly counter, whichever path the send takes.

Deleting your account does not reset it. Sign up again with the same address in the same calendar
month (UTC) — letter case, a `+tag` and, for Gmail, dots do not make it a different one — and the
new account continues from the sends the deleted one had used. The next month starts at zero.

There are **no webhooks and no email logs on BYOC plans.** Your messages are processed entirely
within your AWS account, so we have no access to their delivery events. Read them from your own SES
event destinations and CloudWatch.

<br>

## Shared SES: Managed infrastructure

No AWS account needed. Pick a Managed plan, verify your domain with DNS records, and start sending.
Dashboard logs, delivery webhooks, and automatic bounce, complaint, and suppression handling are
included.

There is **no manual approval step** and no waiting period. Registration is immediate and the
managed path is self-serve from the moment you subscribe.

**What you get:**
- Domain verification through SendFleet's SES, with a guided DNS flow and a downloadable zone file.
- **14 days of email log history** in your dashboard, with per-message delivery status, bounce and
  complaint detail, and the message body.
- **Webhook delivery events**, for whichever stages of the lifecycle you subscribe to per endpoint.
- **Automatic bounce and complaint handling** — suppression, log status, and webhook, without you
  writing any of it.
- **Managed deliverability monitoring** across rolling windows.

**The trade-off:** because all Managed accounts share our sender reputation, we monitor sending
quality and pause sending on accounts that cross the limits. Read
[Bounces, complaints, and suspension](#bounces-complaints-and-suspension) before you scale up.

Email content for Shared SES sends is written to your email log and retained for **14 days**, then
purged automatically. Attachments are staged only for the duration of delivery and are never
written to our database.

### Free plan: the Shared smoke test

Free BYOC can also send through Shared SES — **to your own verified account address and nowhere
else**. It exists so you can check a request end to end through SendFleet's infrastructure before
you set up AWS or pick a plan. **It is a smoke test, not a sending path.**

- **One recipient:** the email address you signed up with, exactly. Any other `to` — a colleague, a
  `+tag` variant, a SendFleet test address, the SES simulator — is refused with
  `403 unauthorized` and nothing is queued. So are a second `to` address and any `cc` or `bcc`.
- **No redirection:** `reply_to` may only be empty, your own address, or the sender address.
  `from_name` may not contain an address, line breaks, or the characters `< > @ , ; "`, and
  `subject` may not contain line breaks.
- **One Shared domain**, verified the same way as on a Managed plan.
- **Same allowance:** smoke tests count towards Free BYOC's 50 emails a month, together with your
  BYOC sends.
- **No webhooks.** Smoke-test sends appear in your email log (14 days, content included — this is
  Shared SES), but no webhook fires for them, even if your account has endpoints left over from a
  previous Managed plan.
- Your BYOC sending on the free plan is unaffected: it still sends to anyone.
- **Zero tolerance for spam reports.** A smoke test only ever reaches your own inbox, so marking one
  as spam is reporting your own mail. The first spam complaint on a smoke test **permanently
  suspends the account** — no warning and no temporary pause first. Delete a smoke test you no
  longer want; do not report it.

To send to anyone else through Shared SES, choose a Managed plan.

<br>

## Plans and pricing

Five plans. Two sending modes. All prices USD, billed monthly in advance.

| Plan | Sends through | Monthly limit | Price |
|---|---|---|---|
| **Free BYOC** | Your AWS SES (BYOC) — plus a Shared smoke test to your own address | 50 (both together) | **Free** |
| **BYOC** | Your AWS SES (BYOC) | Unlimited* | **$9/mo** |
| **Managed Starter** | SendFleet Shared SES | 5,000 | **$5/mo** |
| **Managed Growth** | SendFleet Shared SES | 25,000 | **$15/mo** |
| **Managed Pro** | SendFleet Shared SES | 100,000 | **$59/mo** |

\* *Unlimited = no SendFleet cap. Your own AWS SES sending quota still applies.*

### Compare every plan

| | Free BYOC | BYOC | Managed Starter | Managed Growth | Managed Pro |
|---|---|---|---|---|---|
| **Price** | $0/mo | $9/mo | $5/mo | $15/mo | $59/mo |
| **Sending infrastructure** | Your AWS SES | Your AWS SES | SendFleet SES | SendFleet SES | SendFleet SES |
| **Recipients, subjects, bodies stored by us** | Never for BYOC sends (smoke tests are logged 14 days) | Never | Stored in logs | Stored in logs | Stored in logs |
| **Reputation isolated per-customer** | ✓ | ✓ | — | — | — |
| **Manual account approval** | Not required | Not required | Not required | Not required | Not required |
| **Monthly email volume** | 50 | Unlimited* | 5,000 | 25,000 | 100,000 |
| **REST API access** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Unlimited API keys** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Email log history** | — Privacy by design | — Privacy by design | 14 days | 14 days | 14 days |
| **Domain verification** | Via your SES | Via your SES | Via our SES | Via our SES | Via our SES |
| **Attachments** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Bounce & complaint handling** | Your AWS account | Your AWS account | ✓ | ✓ | ✓ |
| **Shared SES sending** | Smoke test only: your own address | — | ✓ | ✓ | ✓ |
| **Webhook delivery events** | — | — | ✓ | ✓ | ✓ |
| **Support channel** | Email | Email | Priority email | Priority email | Priority email |

### What each plan is for

- **Free BYOC** — Validate your integration end-to-end before committing to anything. 50 emails a
  month, no credit card, no time limit. Includes a Shared smoke test to your own address, so you can
  try a request before setting up AWS; it cannot send to anyone else. Upgrade to $9/month for unlimited sends with no other
  changes to your code.
- **BYOC ($9)** — You want your own sender reputation and no message content retained by us, and 50
  emails a month is too low. Removes the SendFleet cap entirely. Requires a validated IAM role
  first.
- **Managed Starter ($5)** — You want logs, webhooks, and handled bounces without touching AWS at
  all. 5,000 emails a month.
- **Managed Growth ($15)** — The working default for a production app. 25,000 emails a month with
  full observability.
- **Managed Pro ($59)** — 100,000 emails a month, same feature set, priority email support.

### Need something custom?

Enterprise plans with dedicated infrastructure, custom rate limits, uptime commitments, SSO, audit
logs, and a dedicated account manager are available — talk to
**[sales@sendfleet.net](mailto:sales@sendfleet.net)**.

<br>

## DNS and deliverability

Both BYOC and Shared SES domains go through the same DNS verification flow. Records point at
**your** SES for BYOC domains and at SendFleet's for Shared SES domains.

### Required to send

```
TXT    _amazonses.yourdomain.com          <verification_token>
TXT    _sendfleet.yourdomain.com          sendfleet-verification=<value>

CNAME  <token1>._domainkey.yourdomain.com  <token1>.dkim.amazonses.com
CNAME  <token2>._domainkey.yourdomain.com  <token2>.dkim.amazonses.com
CNAME  <token3>._domainkey.yourdomain.com  <token3>.dkim.amazonses.com
```

The `_sendfleet` value is unique to that domain on your account: it proves the domain is yours, not
another account's that added the same name. Domains verified before this record existed do not need
it.

A domain is **send-ready** once both checks pass: ownership verification (both TXT records) **and** DKIM. If
ownership passes but DKIM does not, sending is blocked and your dashboard says so.

### Strongly recommended for inbox placement

These do not gate sending, but they measurably affect deliverability. Verify each one individually
from your dashboard.

```
MX    send.yourdomain.com     10  feedback-smtp.<region>.amazonses.com
TXT   send.yourdomain.com          v=spf1 include:amazonses.com ~all
TXT   _dmarc.yourdomain.com        v=DMARC1; p=none;
```

Notes:

- **No SPF record is needed on your root domain.** DKIM already authenticates the root, and the
  custom MAIL FROM subdomain carries its own SPF record for bounce routing and alignment.
- Set up **one** DMARC record. DMARC is fully disabled when more than one exists at `_dmarc` — if
  you see a conflict warning, remove all but one. Adding a `rua=mailto:` reporting address gives you
  aggregate reports, which is how you catch a problem before your bounce rate does.
- Start with `p=none` to monitor without rejecting mail, then move to `p=quarantine` or `p=reject`
  once DKIM and MAIL FROM are confirmed.
- `send` is the MAIL FROM subdomain on every domain. It is not configurable.

### What the dashboard does for you

- **Verify all records** in one click, or verify ownership, DKIM, MAIL FROM, and DMARC separately.
- **Live DNS checks** per record — it tells you which records are already published, which are
  missing, and which conflict.
- **Downloadable zone file** containing only the records you are still missing, importable
  directly into Cloudflare, Route 53, Namecheap, GoDaddy, and anything else that supports zone-file
  import. Re-importing will not create duplicates.
- **A deliverability score** per domain, plus non-blocking `warnings` on the send response when
  MAIL FROM or DMARC is absent.

DNS propagation can take a few minutes and up to **48 hours**. Check status anytime.

<br>

## Webhooks

Available on **Managed plans only**. Your HTTPS endpoint receives real-time callbacks as messages
move through the delivery lifecycle.

| Event | When |
|---|---|
| `email.sent` | Accepted by the provider and handed off for delivery |
| `email.delivered` | Confirmed delivered to the recipient's mail server |
| `email.delayed` | Temporarily deferred; a retry is scheduled |
| `email.bounced` | Delivery failed permanently |
| `email.complained` | Recipient marked the message as spam |
| `email.rejected` | Rejected before any delivery attempt |
| `email.suppressed` | Suppressed — a prior bounce or complaint put the address on the suppression list |
| `email.failed` | Permanent failure, including rendering failures |

### Verifying the signature

Every request is signed with **HMAC-SHA256** using your endpoint's secret. The signature is sent in
the `X-SendFleet-Signature` header as `sha256=<hex_digest>`, computed over the **raw request body**.

```python
import hashlib, hmac

expected = "sha256=" + hmac.new(secret.encode(), request.body, hashlib.sha256).hexdigest()
if not hmac.compare_digest(expected, request.headers.get("X-SendFleet-Signature", "")):
    return  # reject: not from us
```

```javascript
import { createHmac, timingSafeEqual } from "node:crypto";

const expected = "sha256=" + createHmac("sha256", secret).update(req.rawBody).digest("hex");
const received = req.headers["x-sendfleet-signature"] ?? "";
const ok = expected.length === received.length && timingSafeEqual(Buffer.from(expected), Buffer.from(received));
if (!ok) return; // reject: not from us
```

**Always use a constant-time comparison.** Never use `==` — it is vulnerable to timing attacks.
Never process a webhook without verifying the signature first.

The event name is also sent as `X-SendFleet-Event`, so you can route before parsing the body.

### Payload

Every payload is flat and shares four fields:

```json
{
  "idempotency_key": "8f14e45f-...",
  "ses_message_id": "0100018f...",
  "recipient": "user@example.com",
  "status": "bounced",
  "bounce_type": "Permanent",
  "bounce_subtype": "General",
  "bounced_recipients": ["user@example.com"],
  "timestamp": "2026-09-16T12:00:00.000Z"
}
```

| Field | Description |
|---|---|
| `idempotency_key` | Matches the key in your send response. **This is how you correlate.** |
| `ses_message_id` | The provider's message identifier. |
| `recipient` | The recipient this event is about. A send to several recipients produces one event per recipient. |
| `status` | Terminal or in-flight status for this message. |

Event-specific fields follow. `email.sent` adds `subject`, `mode`, and `attachments`; bounces add
`bounce_type`, `bounce_subtype`, `bounced_recipients`, and `timestamp`; complaints add
`complaint_feedback_type`, `complained_recipients`, and `timestamp`; delays add `delay_type`,
`expiration_time`, and `delayed_recipients`; delivery adds `smtp_response`, `recipients`, and
`processing_time_millis`; rendering failures add `template_name` and `error_message`; and
rejections and suppressions add `reason`.

Note what is **not** in the payload: `message_id` is deliberately absent. Use `idempotency_key`.
Every delivery also carries `User-Agent: SendFleet-Webhook/1.0`, so you can recognise the source in
your logs.

### Endpoint requirements

- Must be **publicly reachable**. HTTPS in production; HTTP is accepted for local testing.
- A signing secret is generated automatically when you create the endpoint and shown **once**. If
  you lose it, delete the endpoint and create a new one.
- Choose which events to subscribe to per endpoint, so you can run a lightweight `email.bounced`
  handler and a verbose everything-handler side by side.
- Pause and resume an endpoint without deleting it.
- Deliveries time out after 5 seconds. **Failed deliveries are not retried** — return `2xx` quickly
  and hand off any real work to a queue of your own, or you will lose events.

<br>

## Error codes

Branch on the machine-readable `error` field. Log or surface `detail` for humans — it is
human-readable and explicitly non-contractual.

```json
{
  "success": false,
  "error": "domain_not_verified",
  "detail": "Domain 'yourdomain.com' is not yet verified. Add both TXT ownership records, then press Verify all records at https://sendfleet.net/dashboard/domains/"
}
```

> **The one exception: `400`.** Validation failures do not use that shape. They return a map of
> field name to messages, so check the status code before reaching for `data.error`.
>
> ```json
> {
>   "to": ["Enter a valid email address."],
>   "attachments": ["File type '.exe' is not permitted for security reasons."]
> }
> ```
>
> A request with no `message`, no `html`, and no `attachments` also lands here, as a non-field
> error: `Provide at least one of 'message', 'html', or 'attachments'.`

| HTTP | `error` | Cause | Fix |
|---|---|---|---|
| 400 | — | Malformed body: bad email, missing `message`/`html`/`attachments`, more than 50 recipients, a repeated recipient, too many attachments, attachment size or type violation | Fix the payload. Do not retry unchanged. |
| 403 | — | Missing, invalid, revoked, or expired API key. This body carries no `error` code, so a bare 403 with no `error` field means the key was the problem. | Fix the key. |
| 403 | `unauthorized` | Account not entitled to this path — email unverified, no active plan granting it, shared sending unavailable, BYOC not yet configured, or a Free plan Shared smoke test addressed to anyone but your own verified address | Verify the account email, check the plan, finish BYOC setup, or send the smoke test to yourself. |
| 403 | `domain_not_registered` | `from_email` domain is not in your account | Add the domain first. |
| 403 | `domain_not_allowed` | API key is scoped to a different set of domains | Use the right key or widen its scope. |
| 403 | `domain_not_verified` | SES ownership or DKIM not yet confirmed | Add the DNS records and verify. |
| 403 | `dkim_not_verified` | All three DKIM CNAMEs not confirmed (Shared SES) | Publish the records, then verify DKIM. |
| 403 | `domain_mode_mismatch` | The domain's sending mode does not match your plan | Add a domain for the mode your plan uses, or switch plans. |
| 403 | `sending_suspended` | Sending stopped on your account: a 24-hour pause that lifts by itself, or a suspension | Check the `detail`: a temporary pause resumes on its own; a suspension needs a support review. |
| 422 | `recipient_suppressed` | *Reserved for contract stability — no longer returned* | Keep any existing handler, but do not expect it to fire. |
| 429 | `usage_limit_exceeded` | Monthly plan quota exhausted, or fewer sends left than the request has recipients | Do not retry. Upgrade, send to fewer recipients, or wait for the quota to reset on the first of the next UTC calendar month. |
| 429 | — | Per-key rate limit exceeded (60/min) | Retry with exponential backoff from 1s, and spread bursts out. |
| 500 | `internal_error` | Unexpected error on our side | Retry once after 2–5s, then alert us with the timestamp. |
| 502 | `queue_error` | Message could not be accepted | Nothing was sent and nothing was counted. Safe to retry: 1s, 2s, 4s, 8s. |
| 502 | `attachment_upload_failed` | Attachment staging failed | Nothing was sent. Safe to retry; check the 25 MB / 37 MB limits. |
| 503 | `byoc_not_configured` | A BYOC configuration problem on our side, not yours | Contact support. Do not retry in a loop. |

### Retry strategy

| Status | Retryable? | Strategy |
|---|---|---|
| 400 | No | Fix the request payload. |
| 403 (no `error` field) | No | Fix the API key. |
| 403 | No | Fix the account or domain configuration. |
| 429 (quota) | No | Upgrade, or wait for the monthly reset. |
| 429 (rate limit) | **Yes** | Exponential backoff from 1s. |
| 500 | **Yes** | One retry after 2–5s, then alert. |
| 502 | **Yes** | Exponential backoff, maximum four attempts. |
| 503 | No | Contact support. |

A monthly-quota `429` is terminal — no amount of waiting clears it. Distinguish the two kinds of
`429` by whether the body carries an `error` field.

`422` is deliberately absent: it is a reserved code that the send path no longer returns. Handle
suppression through the `email.suppressed` webhook and the `suppressed` log status instead.

<br>

## Bounces, complaints, and suspension

On the Shared SES path, SendFleet processes bounce, complaint, and delivery notifications
automatically. You do not have to build any of this.

| Event | What SendFleet does | What you get |
|---|---|---|
| **Hard bounce** | Address added to the suppression list, log marked `bounced` | `email.bounced` webhook |
| **Soft bounce (transient)** | Log marked `bounced`; **no** suppression — the address stays sendable | `email.bounced` webhook |
| **Spam complaint** | Address added to the suppression list, log marked `complained` | `email.complained` webhook |
| **Delivery** | Log marked `delivered` with a timestamp | `email.delivered` webhook |
| **Suppressed address** | Send is accepted, then resolved as suppressed | `email.suppressed` webhook + `suppressed` log status |

A send to a suppressed address is **accepted** with a `200` and resolves to a `suppressed` status
rather than failing at request time. Handle it the way you would have handled the old `422`: mark
the address undeliverable on your side and stop sending to it.

> **Mirror suppressions on your side.** Because the request is accepted before suppression is
> resolved, a retry loop aimed at a suppressed address keeps costing you accepted sends without
> delivering anything. Keep your own suppression list and filter before you call the API — that is
> cheaper and more predictable than discovering it through webhooks.

### How sending quality is measured

Bounce and complaint rates are measured continuously across **three rolling windows** — a short
acute window, 24 hours, and 14 days — each with its own limit. No window is judged below a minimum
send count **and** a minimum failing-message count, because two bounces against ten sends is noise
and we will not act on it. Complaint reports are weighted most heavily, since a spam report is a
judgement about your mail rather than a transient delivery problem.

New accounts are given lower volume floors while their sending history is still short — but not
zero, and not one complaint. A brand-new account is not held to the same volume as a long-running
one, and it is not exempted from the rules either.

**We do not publish the numeric limits.** What keeps you clear of them is straightforward: keep
hard bounce rates low, keep complaint rates very low, and send enough volume that a handful of
transient failures cannot trip a window.

**One exception you should design around:** a spam report classifying your mail as **fraud** or
**virus** acts on its own, whatever the volume or rate around it. Everything else an inbox provider
reports — including an ordinary "mark as spam" button click — counts toward your complaint rate and
is judged by rate like any other signal. If your templates or redirects could be mistaken for
phishing, that is the rule to design around.

**A second exception, on the free plan only:** a spam complaint on a
[Shared smoke test](#free-plan-the-shared-smoke-test) — mail that can only have gone to your own
address — **permanently suspends the account on the first report**, skipping the warning and the
pause below. Free BYOC sending and every paid plan follow the steps below unchanged.

### What happens when you cross a limit

There are three steps, and the first two are reversible.

**1. A warning.** While your bounce or complaint rate is getting close to a limit, we email your
account address with what to check, so there is time to fix the cause. Nothing about your account
changes.

Every step below is emailed to the account address too: the pause (with the time it ends), the
resume, the suspension and any reinstatement. None of those emails states a rate or a limit.

**2. A temporary pause — 24 hours.** Crossing a limit the first time **pauses** API sending for 24
hours. It resumes by itself. The account changes only this:

- **Sending is paused.** `/v1/send` returns `403 sending_suspended`, with a `detail` that says the
  pause is temporary and will lift on its own.
- **Your subscription is untouched.** Your plan keeps running and nothing is cancelled, nothing is
  added to any blocklist, and you do not need to do anything to restart sending.
- **Account changes are frozen while the pause is open** — you cannot add or remove domains, API
  keys, or webhooks, change your BYOC configuration, or delete the account. Revoking an existing API
  key still works; that door stays open on purpose. Everything unfreezes when the pause lapses.

**3. Suspension — and only on a repeat breach.** If your account crosses a limit again after it has
already been paused once, that is a repeat breach and the response becomes permanent:

- **Sending is suspended.** `/v1/send` returns `403 sending_suspended`.
- **Your subscription is ended.** Billing stops and you are not charged again. If sending is later
  reinstated, you will need to start a new subscription to resume sending.
- **Account changes stay frozen.**
- **Reinstating sending does not restore billing.**

Suspension is also what a SendFleet operator will apply directly when there is confirmed abuse,
rather than waiting for the counters to get there first.

**Reinstatement is a support review, not an automated step.** Contact
[support@sendfleet.net](mailto:support@sendfleet.net) to request it, and expect to be asked what
you changed about your list first — a clean list and an honest answer is usually enough to get an
account back.

BYOC accounts build reputation in their own AWS account and are monitored by AWS rather than by us.

### Testing an integration without mailing anyone

SendFleet owns a set of addresses that behave like the outcomes you need to handle. Send to them and
you get a real webhook and a real log entry, without emailing a real person and without touching your
quota or your sending-quality numbers.

| Address | Result |
|---|---|
| `delivered@sendfleet.net` | `email.delivered` webhook, log marked `delivered` |
| `bounce@sendfleet.net` | `email.bounced` webhook, log marked `bounced` |
| `complaint@sendfleet.net` | `email.complained` webhook, log marked `complained` |

`success@sendfleet.net` and `sent@sendfleet.net` are accepted aliases for `delivered@`, and
`bounced@` / `complained@` for the other two, in case your fixture data would rather use those
spellings.

Every send to one of these counts against your monthly allowance like any other send, and is
excluded from your bounce and complaint rates, so testing your webhook handlers cannot get your
account suspended. Each response also carries a `warnings` entry saying so.

Anything at `simulator.amazonses.com` is treated the same way: it counts against your allowance and
never against your rates. Those addresses *are* handed to AWS — the SES simulator accepts and
swallows them, which is useful if you are testing at the SES level.

> **The three test addresses are available on Shared SES only.** Sending to `bounce@sendfleet.net`
> from a BYOC domain returns `403 unauthorized` and is not queued. This is deliberate: the test
> addresses work by SendFleet generating the delivery event itself, and on BYOC your own AWS account
> performs the send with no event destination connected to SendFleet, so there is nothing to
> generate. Allowing it would put a real message into your account addressed to a mailbox that does
> not exist — a real non-delivery report against the sender reputation you own, and no webhook for
> you to test. Simulator addresses do work on BYOC. To test a BYOC integration, send to
> a real address you control.

<br>

## Dashboard

The console at **sendfleet.net/dashboard** covers everything the API does and more — the API has no
endpoints for domains, keys, webhooks, or DNS, all of which you manage in the dashboard.

| Page | What it does |
|---|---|
| **Overview** | Sends used this month against your limit, active key count, current plan and state, and a delivery-outcomes breakdown (delivered / bounced / complained / rejected) for Managed plans. Also a live setup checklist telling you exactly what is still missing. |
| **Email Logs** | 14-day history of every Shared SES message: time, recipient, sender, subject, and status. Open any message for a full status timeline, bounce and complaint detail, and the message body. |
| **API Keys** | Create, name, scope to domains, set an expiry date, revoke, or delete. Tabs for all, active, and revoked-or-expired keys. |
| **Domains** | Add BYOC or Shared SES domains, get the DNS records, download a zone file, verify all records, read live per-record status and your deliverability score. |
| **Webhooks** | Add endpoints, choose events, copy your signing secret, pause, resume, or delete. Includes copy-paste signature verification snippets. |
| **Billing** | Current plan and usage, upgrade, downgrade, cancel, resume, or open the billing portal. |
| **Settings** | Profile, password, account deletion, and the full BYOC IAM role setup wizard with your trust policy and permissions pre-filled. |

Warnings appear where they are actionable — a DKIM failure on a verified domain, a missing domain
altogether, an unverified account email, usage approaching your limit, or a suspended account — each
with a link to the page that fixes it.

> Email logs are view and paginate only. There is no search, filtering, or export today, and no
> manual resend. If you need durable per-message history, use webhooks and store the events
> yourself.

<br>

## Billing behaviour

- Paid plans are billed **monthly in advance**, in USD, excluding applicable taxes.
- **Upgrade** — takes effect as soon as it is confirmed, and the difference for the remaining days
  in your current period is charged on a prorated basis. The full new price applies from your next
  renewal. No waiting for the next cycle.
- **Downgrade** — the lower plan's limits apply as soon as the change is confirmed, also immediately.
  **No refund or credit** is issued for unused days on the prior plan. The next charge is at the
  lower price.
- **Cancel** — cancel at the end of the current period. You keep full access until then, and the
  plan drops to **Free BYOC** (50 BYOC emails/month) at period end.
- **Resume** — restart a paused subscription from the billing page.
- **Billing portal** — manage your payment method, invoices, and cancellation directly.
- **Free BYOC is free forever.** No card, no expiry, no time limit.
- Quotas reset at the start of each **UTC calendar month**. There is no annual billing.

Usage is counted exactly, one slot per recipient (`to`, `cc` and `bcc` alike, as Amazon SES
counts them): every send claims all its slots in the monthly count atomically before it is queued,
so the limit is a hard cut-off even under concurrency, and a send that fails to queue gives its
slots back. Hitting a plan's email limit is not a refundable event. Refund requests must
be raised within 14 days of the charge — 30 days for billing errors; see the
[refund policy](https://sendfleet.net/refund-policy/).

<br>

## Security

**API keys**
- SHA-512 hashed before storage. The raw secret is never persisted and a database breach cannot
  expose it.
- Shown exactly once at creation. A lost key cannot be recovered — revoke it and generate a new one.
- Optional expiry dates, and optional scoping to a specific set of sending domains.
- Revocation is immediate and irreversible. Treat an exposed key as compromised from the moment of
  exposure, not the moment you revoke it.
- Per-key rate limiting at **60 requests/minute**.
- Ten failed authentication attempts from one IP within 60 seconds temporarily blocks that address.

**BYOC role assumption**
- Every send runs on a short-lived session of **up to 15 minutes** on your role. No long-lived
  credential of yours is stored.
- The ExternalId is required in your trust policy's `Condition` block. No ExternalId, no assumption —
  even if someone learns your Role ARN.
- The IAM role name `SendFleet-BYOC-Role` is enforced, case-sensitively, and a role with any other
  name is rejected at setup, so only a conforming role can ever be assumed.
- The role is granted only the SES permissions it needs. Revoke it from your AWS console at any time
  and sending stops immediately.

**Delivery pipeline**
- Every send request is cryptographically signed before it enters the processing pipeline and
  verified again before delivery. Tampered or unsigned messages are rejected and never delivered.

**Webhooks**
- Every delivery request is HMAC-SHA256 signed with your endpoint's secret. Verify with
  `hmac.compare_digest` before processing.
- Endpoint URLs are checked for public routability, so internal and metadata addresses are rejected.

**Account and content privacy**
- BYOC: recipient addresses, subjects, and bodies are never written to our database. Only a monthly
  send counter.
- Shared SES: content is written to your email log and purged after 14 days, except while an account
  is under review, where logs are preserved for as long as the review requires.
- Attachments are held only for the duration of delivery and are then discarded, on every plan. They
  are never written to our database.

<br>

## Limits at a glance

| Limit | Value |
|---|---|
| Requests per minute | 60 per API key (all plans) |
| Failed auth attempts | 10 per IP per 60 seconds |
| Sign-up / login / password-reset attempts | Rate limited per IP over a 5-minute window |
| Recipients per request | 50 across `to`, `cc` and `bcc`, each counted as one send |
| Subject length | 256 characters |
| Sender name length | 156 characters |
| Attachments per message | 20 |
| Attachment size | 25 MB per file, 37 MB per message |
| Request body | 58 MB |
| Sender domains per account | Unlimited |
| API keys per account | Unlimited |
| Email log retention (Shared SES) | 14 days |
| Monthly send limit | 50 / Unlimited / 5,000 / 25,000 / 100,000 by plan |
| DNS propagation | A few minutes, up to 48 hours |
| Test-address sends | Count against the monthly send limit like any other send; never against the bounce or complaint rates |
| Webhook delivery timeout | 5 seconds, no retries |
| IAM role session (BYOC) | Up to 15 minutes |

<br>

## Get started

1. **Create a free account** at **[sendfleet.net/register](https://sendfleet.net/register)** — 50
   BYOC emails a month, no credit card required. Email verification is required before you can
   send.
2. **Connect a sending mode.** BYOC: paste the IAM role ARN from your dashboard into Settings, then
   add a domain. Shared SES: pick a Managed plan, then add a domain.
3. **Add your DNS records** and click **Verify all records**.
4. **Create an API key** and copy it — the secret is shown once.
5. **Send your first email.**

| | |
|---|---|
| Marketing site | **[sendfleet.net](https://sendfleet.net)** |
| Pricing | **[sendfleet.net/pricing/](https://sendfleet.net/pricing/)** |
| API documentation | **[docs.sendfleet.net](https://docs.sendfleet.net)** |
| Quickstart | **[docs.sendfleet.net/quickstart](https://docs.sendfleet.net/quickstart)** |
| Authentication | **[docs.sendfleet.net/authentication](https://docs.sendfleet.net/authentication)** |
| Send endpoint reference | **[docs.sendfleet.net/send-email](https://docs.sendfleet.net/send-email)** |
| Account endpoint | **[docs.sendfleet.net/me](https://docs.sendfleet.net/me)** |
| BYOC setup | **[docs.sendfleet.net/byoc](https://docs.sendfleet.net/byoc)** |
| Shared SES | **[docs.sendfleet.net/shared-ses](https://docs.sendfleet.net/shared-ses)** |
| DNS and deliverability | **[docs.sendfleet.net/dns](https://docs.sendfleet.net/dns)** |
| Webhooks | **[docs.sendfleet.net/webhooks](https://docs.sendfleet.net/webhooks)** |
| Error codes | **[docs.sendfleet.net/errors](https://docs.sendfleet.net/errors)** |
| Plans and limits | **[docs.sendfleet.net/plans](https://docs.sendfleet.net/plans)** |
| Suspension policy | **[docs.sendfleet.net/account](https://docs.sendfleet.net/account)** |

**Contact**

| | |
|---|---|
| Support | **[support@sendfleet.net](mailto:support@sendfleet.net)** |
| Sales & enterprise | **[sales@sendfleet.net](mailto:sales@sendfleet.net)** |
| Billing | **[billing@sendfleet.net](mailto:billing@sendfleet.net)** |
| Security | **[security@sendfleet.net](mailto:security@sendfleet.net)** |
| Privacy & data requests | **[privacy@sendfleet.net](mailto:privacy@sendfleet.net)** |

**Legal** — [Terms & Conditions](https://sendfleet.net/terms-and-conditions/) ·
[Privacy Policy](https://sendfleet.net/privacy-policy/) ·
[Refund Policy](https://sendfleet.net/refund-policy/)

<br>

<p align="center">
  <sub>
    ⚡ <strong>SendFleet</strong> — Your SES or ours. One endpoint. No message content retained for BYOC.
  </sub>
</p>
