<div align="center" style="background-color:#0a0a0a; padding: 44px 20px 40px; border-radius: 12px;">
  <h1 style="color:#f0ede8; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif; font-weight:800; font-size:3em; letter-spacing:-0.03em; margin:0;">
    Send<span style="color:#f5a623;">Fleet</span>
  </h1>
  <p style="color:#8a8480; font-size:0.9em; margin-top:8px;">
    One endpoint. No SDK required. No per‑email markup.
  </p>
</div>

<br>

> **SendFleet is the email API for people who want email to work — not to become a project.**  
> Direct, secure, radically affordable. One endpoint. No SDK required.

<br/>

## 💡 Why SendFleet?

- **Massively lower cost**  
  We sit directly on AWS SES and pass the infrastructure price through to you. No per‑email markup — you pay a flat monthly subscription instead of a hidden premium.

- **One endpoint, any language**  
  A single `POST /api/send/` call works from `curl`, `fetch`, `axios`, or anything that speaks HTTP. No client libraries to maintain, no version hell.

- **Form‑to‑email in 30 seconds**  
  Drop a JavaScript fetch snippet into any static HTML form, and submissions land directly in your inbox. No backend, no serverless function, no per‑message fee.

- **Security you don’t have to think about**  
  API keys are SHA‑512 hashed — we never store the raw secret. Keys can be revoked, renamed, or scoped by date. Every request is verified in constant time.

- **Own your data**  
  Every sent email is logged and visible in your dashboard.

<br/>

## ⚡ One endpoint. That’s it.

```bash
curl -X POST https://sendfleet.net/api/send/ \
  -H "X-API-Key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Jane Doe",
    "email": "jane@example.com",
    "subject": "Welcome!",
    "message": "Thanks for signing up."
  }'
```

**Response (instant, <50ms):**
```json
{
  "success": true,
  "message": "Email queued",
  "message_id": "550e8400-e29b-..."
}
```

The email is delivered in the background.
Every sent email appears in your dashboard with status, timestamp.

<br/>

## 🪶 The missing piece for static sites

```javascript
fetch('https://sendfleet.net/api/send/', {
  method: 'POST',
  headers: {
    'X-API-Key': 'your_api_key',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: form.contactName.value,
    email: form.emailAddress.value,
    subject: 'New contact form submission',
    message: form.userMessage.value
  })
});
```

Paste this into any HTML contact form, and every submission lands straight in your inbox.  
No third‑party service, no per‑submission pricing — just email, done.

<br/>

## 🔐 Security that trusts nobody (not even us)

- **SHA‑512 hashed keys** – we store only a hash, never the raw secret.
- **One‑time reveal** – the full key is shown only at creation.
- **Instant revocation** – remove a key, and all future requests fail immediately.
- **Rate limiting** – per‑key throttling keeps things clean.
- **Constant‑time verification** – every authentication check is hardened against timing attacks.

Your credentials stay yours. We just supply the lock and the door.

<br/>

## 🗣️ Brand voice

We speak like a senior engineer who’s seen enough email infrastructure to know what matters:

- **Direct** – say exactly what the thing does. No filler.
- **Technical** – precise words, no empty jargon.
- **Honest** – when there’s a limitation, we say it plainly.
- **Calm** – error messages are informative, never alarming.

Everything you read here was written by humans, not a committee.

<br/>

## 📬 Get started

Create your free account at **[sendfleet.net](https://sendfleet.net)** – no credit card required.  

<br/>

<p align="center">
  <sub style="color:#8a8480;">
    ⚡ <strong>SendFleet</strong> — Email infrastructure that costs less than a coffee and works before you finish reading this.
  </sub>
</p>
