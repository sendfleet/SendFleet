<div align="center" style="background-color:#0a0a0a; padding: 44px 20px 40px; border-radius: 12px;">
  <h1 style="color:#f0ede8; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif; font-weight:800; font-size:3em; letter-spacing:-0.03em; margin:0;">
    Send<span style="color:#f5a623;">Fleet</span>
  </h1>
  <p style="color:#8a8480; font-size:0.9em; margin-top:8px;">
    Your SES or ours. One API. Zero friction.
  </p>
</div>

<br>

> **SendFleet routes transactional email through your own AWS SES account — or ours.**  
> Zero email content stored on our servers for BYOC. Same REST endpoint either way. No SDK required.

<br>

## Two modes. One endpoint.

SendFleet operates two distinct infrastructure paths. You choose when you add a domain — and the API call looks identical either way.

| | BYOC | Shared SES |
|---|---|---|
| **Sends through** | Your own AWS SES account | SendFleet's managed AWS SES |
| **Email content stored on our servers** | Never | Yes (90 days, in your dashboard) |
| **Sender reputation** | Isolated to your AWS account | Shared pool (Growth/Pro) |
| **AWS account required** | Yes (yours, out of sandbox) | No |
| **Manual approval required** | No | Yes (1–2 business days) |
| **Email log dashboard** | — (use your SES/CloudWatch) | Full history |
| **Webhook delivery events** | — | ✓ |
| **Price** | Free (50/mo) or $9/mo unlimited | $15/mo (25k) or $59/mo (100k) |

<br>

## BYOC — Bring Your Own SES

The default path. You connect your AWS IAM role; we assume it with a 15-minute STS credential to call `ses:SendEmail` in your account. Your email content never touches our database.

**What you need:**
- An AWS account with SES approved for production access (out of sandbox)
- A domain you control
- Permission to create IAM roles

**Setup in three steps:**

1. Create an IAM role named `SendFleet-BYOC-Role` in your AWS account with the trust policy shown in your dashboard settings. The trust policy includes your unique ExternalId — this prevents the [confused deputy attack](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html).
2. Paste the Role ARN in **Dashboard → Settings → BYOC**. We validate sandbox status and IAM permissions in real time.
3. Add a sending domain. We register the SES identity in *your* account and show you the DNS records to add.

```bash
curl -X POST https://sendfleet.net/api/send/ \
  -H "X-API-Key: your_api_key" \
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
  "message_id": "550e8400-e29b-41d4-a716-...",
  "mode": "byoc"
}
```

Email content exists only in the SQS message (consumed and discarded by our relay) and your own SES account logs. We store one number: your monthly send count.

<br>

## Shared SES — Managed infrastructure

No AWS account needed. Verify your domain with DNS records, get manual approval (to protect the shared sending pool), and you're sending. Full email logs, webhook events, and bounce/complaint handling included.

Approval is required because all Growth and Pro accounts share our SES sender reputation. We review every account before granting access — typically within 1–2 business days.

```json
{
  "success": true,
  "message": "Email queued for delivery.",
  "message_id": "9f8e7d6c-5b4a-...",
  "mode": "shared"
}
```

<br>

## Pricing

| Plan | Infrastructure | Monthly limit | Price |
|---|---|---|---|
| **Starter** | Your AWS SES (BYOC) | 50 emails | Free |
| **BYOC** | Your AWS SES (BYOC) | Unlimited* | $9/mo |
| **Growth** | SendFleet Shared SES | 25,000 | $15/mo |
| **Pro** | SendFleet Shared SES | 100,000 | $59/mo |

*Unlimited = no SendFleet cap. Your AWS SES daily quota applies.

<br>

## Security

**API keys**
- SHA-512 hashed before storage. We never hold the raw secret.
- Shown exactly once at creation. Lost key? Revoke and generate a new one.
- Revocation propagates within ~2 minutes via Redis cache.
- Optional expiry dates. Per-key rate limiting at 60 req/min.

**BYOC role assumption**
- Every send uses a fresh 15-minute STS credential.
- ExternalId is required in your trust policy's `Condition` block — no ExternalId, no assumption, even if someone learns your Role ARN.
- The IAM role name `SendFleet-BYOC-Role` is enforced on every assumption call. Our relay can only assume roles with that exact name.

**Queue payloads**
- Every SQS message is HMAC-SHA256 signed. Our Lambda workers reject unsigned or tampered messages before processing.

**Webhooks (Shared SES)**
- Every webhook request is HMAC-SHA256 signed with your endpoint's secret. Verify with `hmac.compare_digest` before processing.

<br>

## DNS records

Both BYOC and Shared SES domains go through the same DNS verification flow. Minimum required to send:

```
TXT    _amazonses.yourdomain.com          <verification_token>
CNAME  <token1>._domainkey.yourdomain.com  <token1>.dkim.amazonses.com
CNAME  <token2>._domainkey.yourdomain.com  <token2>.dkim.amazonses.com
CNAME  <token3>._domainkey.yourdomain.com  <token3>.dkim.amazonses.com
```

Strongly recommended for deliverability (verify individually from your dashboard):

```
TXT   yourdomain.com              v=spf1 include:amazonses.com ~all
MX    send.yourdomain.com         10 feedback-smtp.<region>.amazonses.com
TXT   send.yourdomain.com         v=spf1 include:amazonses.com ~all
TXT   _dmarc.yourdomain.com       v=DMARC1; p=none;
```

<br>

## Error codes

All errors follow a consistent shape:

```json
{
  "success": false,
  "error": "domain_not_verified",
  "detail": "Domain 'yourdomain.com' is not yet verified by SES. Add the TXT record and click Verify in the dashboard."
}
```

| HTTP | `error` | Cause |
|---|---|---|
| 400 | — | Malformed request body |
| 401 | — | Missing, invalid, or revoked API key |
| 403 | `unauthorized` | Account not approved or email unverified |
| 403 | `domain_not_registered` | `from_email` domain not in your account |
| 403 | `domain_not_verified` | SES ownership not confirmed |
| 403 | `dkim_not_verified` | DKIM CNAMEs not confirmed |
| 403 | `domain_mode_mismatch` | Domain's mode doesn't match your plan |
| 403 | `sending_suspended` | Account suspended (bounce/complaint threshold) |
| 422 | `recipient_suppressed` | Address on suppression list |
| 429 | `usage_limit_exceeded` | Monthly quota exhausted |
| 429 | — | Rate limit exceeded (60 req/min) |
| 502 | `queue_error` | SQS temporarily unavailable — retry with backoff |
| 503 | `byoc_not_configured` | BYOC infrastructure misconfiguration — contact support |

<br>

## Bounce and complaint handling (Shared SES)

SendFleet automatically processes SNS bounce and complaint notifications from our SES configuration set.

- **Hard bounce** → recipient added to suppression list, log marked `failed`, `email.bounced` webhook fired
- **Spam complaint** → recipient added to suppression list, `email.complained` webhook fired
- **Delivery** → log marked `sent` with timestamp, `email.delivered` webhook fired

Accounts are monitored against a rolling 7-day bounce rate and complaint rate window. Breaching either threshold triggers automatic sending suspension. Reinstatement requires a support review — it is never automatic.

<br>

## Get started

Create your free account at **[sendfleet.net](https://sendfleet.net)** — no credit card required.

Read the full API docs at **[sendfleet.net/docs/](https://sendfleet.net/docs/)**.

<br>

<p align="center">
  <sub>
    ⚡ <strong>SendFleet</strong> — Your SES or ours. One endpoint. Zero content stored for BYOC.
  </sub>
</p>
