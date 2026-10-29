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
> infrastructure. Both modes use the same sending endpoint and request fields, with mode-specific
> limits and response fields. On BYOC, connecting your AWS account is one click, SendFleet sets up
> delivery tracking in it for you, and message bodies are never written to our database. No SDK
> required.

<br>

## Contents

- [Two modes. One endpoint.](#two-modes-one-endpoint)
- [Quickstart](#quickstart)
- [API base URL and endpoints](#api-base-url-and-endpoints)
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
a BYOC domain sends through **your** AWS SES account, and a Shared SES domain uses managed sending.

> **Before you switch plans, read this.** A domain's sending mode is permanent. If you move from a
> BYOC plan to a Managed plan — or the other way — your existing domains keep their original mode
> and your sends start failing with `403 domain_mode_mismatch`. Add fresh domains for the new mode
> and retire the old ones. Switching is fully self-serve and takes minutes, not a support ticket.

| | BYOC | Shared SES |
|---|---|---|
| **Sends through** | Your own AWS SES account | SendFleet's managed SES |
| **Recipients, subjects, bodies stored on our servers** | Bodies never. Recipients and subjects only while email logs are on (the default; 14 days) | Yes — 14 days, in your dashboard |
| **Sender reputation** | Isolated to your AWS account | Shared pool across Managed accounts |
| **AWS account required** | Yes (yours, out of sandbox) | No |
| **Manual approval required** | No | No |
| **Monthly send limit** | 3,000 (Free), or 20,000 included then pay as you go up to 1,000,000 ($5 + usage) | 5,000 / 25,000 / 100,000 |
| **Email log dashboard** | Optional — 14 days, metadata only | Full 14-day history |
| **Delivery outcomes** | Tracked from your own SES events | Managed for you |
| **Webhook delivery events** | ✓ (BYOC plan) | ✓ |
| **Bounce and complaint protection** | ✓ — levels you choose | Managed for you |
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
  "idempotency_key": "8f14e45fceea167a5a36dedd4bea2543",
  "mode": "byoc"
}
```

`idempotency_key` is there because delivery tracking is on by default; a BYOC send with tracking
off has none.

A `200 OK` means the message was **accepted and queued**, not delivered. Use [webhooks](#webhooks)
or your email log to track delivery — on BYOC, both come from your own SES account's events once
delivery tracking is on.

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

## API base URL and endpoints

**Base URL: `https://api.sendfleet.net/v1`**

| Method | URL | Purpose |
|---|---|---|
| `POST` | `https://api.sendfleet.net/v1/send` | Send an email |
| `GET` | `https://api.sendfleet.net/v1/me` | Read account details and current-month usage |

Authenticate both endpoints with `X-API-Key`. Sending requires a JSON body and
`Content-Type: application/json`; the account endpoint requires no request body. API responses
are JSON. Manage domains, API keys, webhooks and billing in the dashboard.

<br>

## Send endpoint

**`POST https://api.sendfleet.net/v1/send`**

Every request sends `Content-Type: application/json` and your key in the `X-API-Key` header.
Request bodies are JSON; responses are always JSON. This is the only sending endpoint.

### Request fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `to` | email or email[] | **Yes** | One address, or a list. `to`, `cc` and `bcc` together hold at most **50** recipients (the Amazon SES per-message limit), and no address may appear twice across them. Each sending path has its own per-message rule, and on Shared SES `to` carries exactly one address — see [Recipients per message](#recipients-per-message). |
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

### Recipients per message

**Recipient limits depend on the sending mode.**

| Path | Recipients in `to` | Recipients per message |
|---|---|---|
| **BYOC** — your own AWS SES account | A string or a list | **50** across `to`, `cc` and `bcc`, from your first send. That is Amazon SES's own per-message limit |
| **Shared SES** — SendFleet's account | **Exactly one** | **5** across `to`, `cc` and `bcc` — the one in `to`, plus up to four in `cc` and `bcc` |

If you need more than five recipients in one message, use BYOC. Your own SES sending quotas
also apply to BYOC sends.

**On Shared SES the message is addressed to exactly one person.** `to` carries a single address, and
the rest of the allowance rides in `cc` and `bcc`:

```json
{
  "from_email": "hello@yourdomain.com",
  "to": "primary@example.com",
  "cc": ["second@example.com", "third@example.com"],
  "bcc": ["fourth@example.com", "fifth@example.com"],
  "subject": "Order update",
  "message": "Your order has shipped."
}
```

That is five recipients, each counted as one send, and it is the most one message on the shared path
may carry. More than one address in `to` is refused with `403 recipient_limit_exceeded`.
A single address can be a string or a one-item list. Nothing is sent or counted against your
allowance when the request is refused.

**The limit is per message**, not per day or per month, and it counts `to`, `cc` and `bcc` together.
Going over it is refused with `403 recipient_limit_exceeded`: nothing is sent, and nothing is counted
against your monthly allowance.

### Several recipients

One request sends **one message** to everyone in `to`, `cc` and `bcc`. This example uses BYOC:

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
- **Shared SES supports one address in `to` and up to five recipients in total.** Exceeding
  either Shared SES limit returns `403 recipient_limit_exceeded`. More than 50 recipients
  in any mode fails request validation with `400`. See [Recipients per message](#recipients-per-message).
- **On Shared SES, and on BYOC with delivery tracking on, every recipient is its own entry** in your
  email log (on BYOC, while email logs are on) and gets its own webhooks, all under the send's one
  `idempotency_key`, with the address in `recipient`.
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
| `idempotency_key` | Shared SES, and BYOC with delivery tracking on | The correlation key for this send. The same value appears on every webhook fired for it. Absent on a BYOC send while tracking is off. |
| `mode` | both | `"byoc"` or `"shared"` — the sending mode selected for this request, not a delivery confirmation. |
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

Every accepted send is assigned a unique key, and whenever delivery is tracked (always on Shared SES,
and on BYOC with delivery tracking on) that key follows the message all the way to delivery.

- **On Shared SES**, it is returned in the response as `idempotency_key` and appears on every
  webhook fired for that send. **Store it** — it is how you tie a response to its delivery events.
  A send to several recipients has one key and fires each event once per recipient, so match on
  `idempotency_key` and `recipient` together.
- **On BYOC with delivery tracking on**, it works the same way: the key is returned as
  `idempotency_key`, travels with the message through your own SES account, and is on every webhook
  and log entry for it. With tracking off there is nothing to correlate against; keep the
  `message_id` for your own records.

`message_id` never appears on a webhook, so for webhook correlation `idempotency_key` is the only
field to use.

> **Retry safely.** These keys are correlation handles, not de-duplication tokens — SendFleet does
> not suppress a second identical call. If a send times out or you receive no response, check delivery
> state before retrying: the message may already have been accepted. A confirmed `502 queue_error`
> or `502 attachment_upload_failed` means nothing was sent and is safe to retry with backoff.

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
| `byoc_usage_current_month` | BYOC sends this calendar month (UTC). **On Free BYOC this is the plan's one counter** and includes Shared smoke-test sends (see [Free plan: the Shared smoke test](#free-plan-the-shared-smoke-test)); it is the number checked against the 3,000. On BYOC it is the number checked against the monthly cap you set (default 100,000, up to 1,000,000). |
| `shared_usage_current_month` | Shared sends this calendar month. On a Managed plan this is exactly the number checked against your plan limit. On Free BYOC it only breaks out how many of your sends were smoke tests. |
| `shared_delivered_current_month` | Feeds your delivery rate. |
| `shared_bounced_current_month` | Feeds your bounce rate. |
| `shared_complained_current_month` | Feeds your complaint rate. |
| `shared_rejected_current_month` | Permanent rejections. |
| `byoc_configured` | Whether your AWS account is connected and SendFleet can use it. |
| `verified_byoc_domains` / `verified_shared_domains` | Verified domains per mode. |

A month with no activity returns zero rather than omitting the field. Previous months are not
included. `GET /v1/me` counts against the same 60 requests/minute limit as sending — call it at
startup or on a schedule, not before every individual email.

The response is a snapshot, not authorization or a quota reservation for a future send. Always
handle the send response: key scope, domain readiness and sending restrictions also apply.

`GET https://api.sendfleet.net/health` is an unauthenticated liveness endpoint. It does not confirm
account readiness or delivery status.

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
| **Revoke** | Once confirmed, subsequent API requests using the key are rejected. Irreversible; messages already accepted for delivery are not cancelled. The record is retained as an audit trail. |
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

BYOC delivers your email through **your own AWS SES account**. Your sender reputation is entirely your
own, message bodies never land in SendFleet's database, and SendFleet does the AWS work for you:
connecting, delivery tracking, production access, and protecting your reputation.

**What you need:** an AWS account you can create an IAM role in, and a domain you control. A brand
new SES account is fine — SendFleet requests production access for it.

### Connect in two steps

Open **Dashboard → AWS**.

1. **Create the role with one click.** The button opens AWS CloudFormation with a ready-made stack
   that creates one IAM role, `SendFleet-BYOC-Role`. Tick the IAM acknowledgement and press
   **Create stack**. (Prefer to do it by hand? The same page shows the exact trust and permissions
   policies.)
2. **Paste the role ARN** from the stack's **Outputs** tab and press **Connect**.

That is all. SES production access is separate in every AWS region, so SendFleet checks every
region and sends from the one where your account is out of the sandbox (or from a region you pick,
for example to keep mail in the EU). It validates the role, reads your SES account's health,
switches on your suppression list and Auto Validation, and sets up delivery tracking — all in your
account, automatically. Then add a sending domain and publish its DNS records.

The stack opens in US East (N. Virginia) whatever region you send from: an IAM role is global.

### What the role allows

Sending mail; registering and checking your sending domains; reading your SES account's health and
reputation; requesting production access and switching on the account-level suppression list; and
creating, repairing and removing SendFleet's own tracking resources — one configuration set and one
SNS topic, both named `sendfleet-*`. Nothing else: it cannot read your mail, change IAM, or touch any
other resource.

Only SendFleet's own sending and management services can use the role, and only with your
**ExternalId** — 48 hex characters, generated for your account and never changed. It prevents the
[confused deputy attack](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html):
without it, anyone who learned your Role ARN could ask us to use it. **Never remove the ExternalId
condition.** Delete the CloudFormation stack at any time to revoke SendFleet's access.

### Delivery tracking and what we store

Choose one, and change it whenever you like:

| Mode | What you get | What SendFleet stores |
|---|---|---|
| **Metrics, webhooks and email logs** (default) | Everything below, plus a 14-day email log | One row per recipient: address, status, timestamps, bounce and complaint detail, and the subject unless you turn subjects off. **Never the body.** |
| **Metrics and webhooks** | Delivery and bounce rates, webhooks, reputation protection | Hourly counts only |
| **Send only** | Sending | Only the monthly send counter |

Tracking works by a configuration set in your SES account that reports events to SendFleet through
an SNS topic. **AWS bills your account for each event published**, so you choose:

- **Always on:** bounces, complaints and rejections — protection depends on them, and they are rare.
- **Your choice:** deliveries (about one event per recipient, most of the cost) and delays.
- **Free:** "sent". SendFleet records it itself when SES accepts the message; your account never
  publishes it.

With email logs on, every send appears in your log as **Queued** the moment it is accepted, and
turns **Failed**, with the reason, if it never reaches SES.

If the configuration set or topic is changed or deleted, SendFleet repairs it — unless you turned
**automatic repair** off, in which case it only tells you. If they are removed again soon after a
repair, SendFleet assumes you meant it, stops, and emails you. Your mail is never held up by
tracking: if the configuration set is missing, the message is still sent, just untracked.

**Every change SendFleet makes in your account is listed** under *Activity in your AWS account*,
marked as yours or automatic. Your AWS CloudTrail shows the same calls.

### Account health and production access

Dashboard → AWS shows your SES account live: production access, AWS's view of your standing, your
24-hour quota and sending rate, and the bounce and complaint rates AWS itself measures.

**Still in the SES sandbox?** Pick the region, fill in your website and the kind of mail you send, and
SendFleet files the production access request with AWS for you. It also switches on your account-level suppression
list, so the request truthfully describes how bounces and complaints are handled. AWS answers by
email, usually within a day.

### Reputation protection

AWS reviews an SES account once its bounce rate reaches 5% or its complaint rate 0.1%. SendFleet
watches your rates over 24 hours and 7 days and **pauses your sending before AWS has to**. Choose
**High**, **Medium** (default), **Low** or **Off** — the dashboard shows each level's exact lines.
You are emailed a warning first. A pause makes `/v1/send` return `403 sending_suspended` for your BYOC
domains until you review the bounces and press **Resume** in Dashboard → AWS.

### If access stops working

If the role is deleted or its trust policy changed, SendFleet confirms the failure, stops accepting
BYOC sends (`403 unauthorized`, with a message saying so), and emails you once. Messages already
queued are retried briefly, then dropped rather than retried for days. Recreate the role and press
**Reconnect**; SendFleet also notices on its own when access returns.

### Regions

`us-east-1`, `us-east-2`, `us-west-2`, `ca-central-1`, `sa-east-1`, `eu-west-1`, `eu-west-2`,
`eu-central-1`, `eu-north-1`, `ap-south-1`, `ap-southeast-1`, `ap-southeast-2`, `ap-northeast-1`.

### BYOC limits and billing

- **Free BYOC** — 3,000 emails per month, with delivery tracking and email logs. No webhooks.
- **BYOC ($5/month + usage)** — 20,000 emails and 20,000 logged emails included each calendar month
  (UTC). Beyond that, **$0.10 per 1,000 emails sent** and **$0.10 per 1,000 emails logged** (only
  with email logs on), rounded up to the next 1,000 and added to your next renewal invoice. Webhooks
  included. Your own AWS SES charges are billed by AWS, separately.
- **What counts** — only usage while you are on the BYOC plan. Sends on the Free plan earlier in the
  month, sends after the plan ends and emails Amazon SES did not accept are never charged. A month is
  charged once; if your subscription is set to end, that month's usage so far is charged just before
  it ends.
- **Your monthly cap** — set it in Dashboard → AWS (default 100,000, up to 1,000,000). Sends past it
  return `429 usage_limit_exceeded`, so a bug in your code can never run up a bill. Your own AWS SES
  quota also applies; whichever is lower is the limit you meet first. Need more than 1,000,000?
  Email support@sendfleet.net.

The Free BYOC allowance also covers the [Shared smoke test](#free-plan-the-shared-smoke-test): one
monthly counter, whichever path the send takes.

Deleting your account does not reset it. Sign up again with the same address in the same calendar
month (UTC) — letter case, a `+tag` and, for Gmail, dots do not make it a different one — and the
new account continues from the sends the deleted one had used. The next month starts at zero.

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
  `403 unauthorized` and nothing is queued. So is any `cc` or `bcc`. A second address in `to` is
  refused even earlier, by the shared path's one-address rule, with
  `403 recipient_limit_exceeded` — the same answer, under a different code.
- **No redirection:** `reply_to` may only be empty, your own address, or the sender address.
  `from_name` may not contain an address, line breaks, or the characters `< > @ , ; "`, and
  `subject` may not contain line breaks.
- **One Shared domain**, verified the same way as on a Managed plan.
- **Same allowance:** smoke tests count towards Free BYOC's 3,000 emails a month, together with your
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
| **Free BYOC** | Your AWS SES (BYOC) — plus a Shared smoke test to your own address | 3,000 (both together) | **Free** |
| **BYOC** | Your AWS SES (BYOC) | 20,000 included, then pay as you go up to 1,000,000* | **$5/mo + usage** |
| **Managed Starter** | SendFleet Shared SES | 5,000 | **$5/mo** |
| **Managed Growth** | SendFleet Shared SES | 25,000 | **$15/mo** |
| **Managed Pro** | SendFleet Shared SES | 100,000 | **$59/mo** |

\* *$0.10 per 1,000 emails beyond 20,000, and $0.10 per 1,000 logged emails beyond 20,000. Your own monthly cap and your AWS SES sending quota also apply; whichever is lower is the limit you meet first.*

### Compare every plan

| | Free BYOC | BYOC | Managed Starter | Managed Growth | Managed Pro |
|---|---|---|---|---|---|
| **Price** | $0/mo | $5/mo + usage | $5/mo | $15/mo | $59/mo |
| **Sending infrastructure** | Your AWS SES | Your AWS SES | SendFleet SES | SendFleet SES | SendFleet SES |
| **Recipients, subjects, bodies stored by us** | Bodies never; recipients and subjects only with email logs on (smoke tests are logged 14 days) | Bodies never; recipients and subjects only with email logs on | Stored in logs | Stored in logs | Stored in logs |
| **Reputation isolated per-customer** | ✓ | ✓ | — | — | — |
| **Manual account approval** | Not required | Not required | Not required | Not required | Not required |
| **Monthly email volume** | 3,000 | 20,000 included, up to 1,000,000* | 5,000 | 25,000 | 100,000 |
| **REST API access** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Unlimited API keys** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Email log history** | Optional, 14 days, metadata only | Optional, 14 days, metadata only | 14 days | 14 days | 14 days |
| **Domain verification** | Via your SES | Via your SES | Via our SES | Via our SES | Via our SES |
| **Attachments** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Bounce & complaint handling** | Tracking and protection in your AWS account | Tracking and protection in your AWS account | ✓ | ✓ | ✓ |
| **Shared SES sending** | Smoke test only: your own address | — | ✓ | ✓ | ✓ |
| **Webhook delivery events** | — | ✓ | ✓ | ✓ | ✓ |
| **Support channel** | Email | Email | Priority email | Priority email | Priority email |

### What each plan is for

- **Free BYOC** — Validate your integration end-to-end before committing to anything. 3,000 emails a
  month, no credit card, no time limit. Includes a Shared smoke test to your own address, so you can
  try a request before setting up AWS; it cannot send to anyone else. Upgrade to the $5 BYOC plan
  with no other changes to your code.
- **BYOC ($5 + usage)** — Your own SES and sender reputation, with SendFleet doing the AWS work:
  delivery tracking, webhooks, email logs and bounce protection. 20,000 emails included each month,
  then $0.10 per 1,000, with a cap you set. Requires a connected AWS account first.
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

The `_sendfleet` value belongs to that domain on your account, and stays the same: remove the domain
and add it again, or add it in the other sending mode, and the record you published still works. It
proves the domain is yours, not another account's that added the same name. Domains verified before this record existed do not need
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

Available on the **BYOC plan** and the **Managed plans**. Your HTTPS endpoint receives real-time
callbacks as messages move through the delivery lifecycle. On BYOC they are built from your own SES
account's events (delivery tracking must be on) and carry `"mode": "byoc"`; the payload is otherwise
identical.

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
| 403 | `unauthorized` | Account not entitled to this path — email unverified, no active plan granting it, shared sending unavailable, BYOC not yet configured or its AWS access revoked, or a Free plan Shared smoke test addressed to anyone but your own verified address | Verify the account email, check the plan, finish or reconnect BYOC in Dashboard → AWS, or send the smoke test to yourself. |
| 403 | `recipient_limit_exceeded` | The message is shaped or sized in a way this account may not send on the path it took. 50 across `to`, `cc` and `bcc` on BYOC, where `to` takes a string or a list. On Shared SES, `to` must carry exactly one address and the message may carry 5 in total. | Put one address in `to` and move the rest to `cc`/`bcc`, or send through your own AWS SES account — the `detail` says which. |
| 403 | `domain_not_registered` | `from_email` domain is not in your account | Add the domain first. |
| 403 | `domain_not_allowed` | API key is scoped to a different set of domains | Use the right key or widen its scope. |
| 403 | `domain_not_verified` | SES ownership or DKIM not yet confirmed | Add the DNS records and verify. |
| 403 | `dkim_not_verified` | All three DKIM CNAMEs not confirmed (Shared SES) | Publish the records, then verify DKIM. |
| 403 | `domain_mode_mismatch` | The domain's sending mode does not match your plan | Add a domain for the mode your plan uses, or switch plans. |
| 403 | `sending_suspended` | Sending stopped on your account: a 24-hour pause that lifts by itself, a suspension, or — on BYOC — your own reputation protection | Check the `detail`: a temporary pause resumes on its own; a protection pause resumes when you press Resume in Dashboard → AWS; a suspension needs a support review. |
| 422 | `recipient_suppressed` | *Reserved for contract stability — no longer returned* | Keep any existing handler, but do not expect it to fire. |
| 429 | `usage_limit_exceeded` | Monthly plan quota or your own BYOC monthly cap exhausted, or fewer sends left than the request has recipients | Do not retry. Upgrade (BYOC is the largest plan; above it, email support@sendfleet.net), send to fewer recipients, or wait for the quota to reset on the first of the next UTC calendar month. |
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

### Protecting sending quality

SendFleet monitors bounce and complaint activity to protect deliverability on Shared SES.
**Enforcement thresholds are not published.** Keep recipient lists accurate, send only expected
mail, and remove addresses that hard-bounce or report spam. Use dashboard logs and delivery
webhooks to identify problems and act promptly on account notices.

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

BYOC accounts build reputation in their own AWS account, which AWS reviews. On BYOC, your own
[reputation protection](#reputation-protection) watches the same bounce and complaint rates and
pauses your sending before AWS has to.

### Testing an integration without mailing anyone

SendFleet owns a set of addresses that behave like the outcomes you need to handle. Send to them and
you get a real webhook and a real log entry without emailing a real person. Test sends count
against your monthly allowance but not your bounce or complaint rates.

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
> performs the send, so there is no point at which SendFleet could stand in for it. Allowing it would
> put a real message into your account addressed to a mailbox that does not exist — a real
> non-delivery report against the sender reputation you own. Simulator addresses do work on BYOC:
> with delivery tracking on, a simulated bounce or complaint reaches your email log and, on the BYOC
> plan, your webhooks. AWS leaves simulator events out of its view of your account, but your
> SendFleet reputation protection counts them like any other, so keep simulator tests to a handful.
> To test deliverability end to end, send to a real address you control.

<br>

## Dashboard

The console at **sendfleet.net/dashboard** covers everything the API does and more — the API has no
endpoints for domains, keys, webhooks, or DNS, all of which you manage in the dashboard.

| Page | What it does |
|---|---|
| **Overview** | Sends used this month against your limit, active key count, current plan and state, and a delivery-outcomes breakdown (delivered / bounced / complained / rejected) for Managed plans. Also a live setup checklist telling you exactly what is still missing. |
| **Email Logs** | 14-day history of every Shared SES message, and of every BYOC message while email logs are on: time, recipient, sender, subject, and status. Open any message for a full status timeline and bounce and complaint detail, plus the message body on Shared SES (BYOC bodies are never stored). |
| **API Keys** | Create, name, scope to domains, set an expiry date, revoke, or delete. Tabs for all, active, and revoked-or-expired keys. |
| **Domains** | Add BYOC or Shared SES domains, get the DNS records, download a zone file, verify all records, read live per-record status and your deliverability score. |
| **Webhooks** | Add endpoints, choose events, copy your signing secret, pause, resume, or delete. Includes copy-paste signature verification snippets. |
| **Billing** | Current plan and usage, upgrade, downgrade, cancel, resume, or open the billing portal. |
| **AWS** | BYOC: one-click connection, live SES account health, production access requests, delivery tracking, reputation protection, suppression list, usage and your monthly cap. |
| **Settings** | Profile, password and account deletion. |

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
  plan drops to **Free BYOC** (3,000 BYOC emails/month) at period end.
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
- The role is granted only what SendFleet uses: sending, your domains, account health, and its own
  `sendfleet-*` tracking resources — no IAM, no other resources, no mail content. Only SendFleet's
  sending and management services can use it. Delete the CloudFormation stack to revoke access.

**Delivery pipeline**
- Every send request is cryptographically signed before it enters the processing pipeline and
  verified again before delivery. Tampered or unsigned messages are rejected and never delivered.

**Webhooks**
- Every delivery request is HMAC-SHA256 signed with your endpoint's secret. Verify with
  `hmac.compare_digest` before processing.
- Endpoint URLs are checked for public routability, so internal and metadata addresses are rejected.

**Account and content privacy**
- BYOC: message bodies are never written to our database. With email logs on, recipient addresses,
  subjects (unless you turn them off) and delivery status are kept for 14 days; with metrics only,
  hourly counts; with tracking off, only a monthly send counter.
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
| Recipients per request | 50 across `to`, `cc` and `bcc` on BYOC, where `to` takes a string or a list · on Shared SES, exactly **one** address in `to` and **5** in total — the one plus up to four in `cc`/`bcc`. Every recipient counts as one send |
| Subject length | 256 characters |
| Sender name length | 156 characters |
| Attachments per message | 20 |
| Attachment size | 25 MB per file, 37 MB per message |
| Request body | 58 MB |
| Sender domains per account | Unlimited |
| API keys per account | Unlimited |
| Email log retention | 14 days (BYOC: only with email logs on) |
| Monthly send limit | 3,000 / 20,000 included then up to 1,000,000 / 5,000 / 25,000 / 100,000 by plan |
| DNS propagation | A few minutes, up to 48 hours |
| Test-address sends | Count against the monthly send limit like any other send; never against the bounce or complaint rates |
| Webhook delivery timeout | 5 seconds, no retries |
| IAM role session (BYOC) | Up to 15 minutes |

<br>

## Get started

1. **Create a free account** at **[sendfleet.net/register](https://sendfleet.net/register)** — 3,000
   BYOC emails a month, no credit card required. Email verification is required before you can
   send.
2. **Connect a sending mode.** BYOC: in Dashboard → AWS, create the role with one click and paste its
   ARN, then add a domain. Shared SES: pick a Managed plan, then add a domain.
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
    ⚡ <strong>SendFleet</strong> — Your SES or ours. One endpoint. No message bodies retained for BYOC.
  </sub>
</p>
