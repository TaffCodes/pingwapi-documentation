# PingWapi WhatsApp API — Developer Documentation

**Connect your existing WhatsApp number once by scanning a QR code, then send and receive WhatsApp messages from your own code through a simple REST API.**

No Meta Business verification. No template approvals. No per-message fees — every plan includes unlimited monthly messages at a flat rate.

- **Base URL:** `https://api.pingwapi.com`
- **Auth:** `Authorization: Bearer <API_KEY>` (per-instance keys, issued once at creation)
- **Message types:** text · image · video · audio (incl. voice notes) · document · sticker · location · contact cards
- **Webhooks:** signed HMAC-SHA256 incoming-message events with retries

---

## Table of Contents

1. [What you can build](#what-you-can-build)
2. [Quick Start](#quick-start)
3. [Authentication](#authentication)
4. [Instances & Phones](#instances--phones)
5. [Sending Messages](#sending-messages)
6. [Check Number](#check-number)
7. [Scheduled Messages](#scheduled-messages)
8. [Webhooks](#webhooks)
9. [Message History](#message-history)
10. [Media Upload](#media-upload)
11. [Rate Limits & Quotas](#rate-limits--quotas)
12. [Error Codes](#error-codes)
13. [Code Samples by Language](#code-samples-by-language)

---

## What you can build

PingWapi gives your product a WhatsApp channel without Meta onboarding. If you
can make an HTTP request, you can send a message. Here are some of the things
teams build with it.

| Use case | What it looks like |
|---|---|
| **OTP & account verification** | Send one-time codes for signup, password reset or sensitive actions. Fast delivery, no template review, easy retries. |
| **Orders, receipts & delivery updates** | Confirm an order, announce dispatch, attach the invoice as a PDF and send a delivery confirmation. |
| **Payment & billing reminders** | Payment confirmations, renewal reminders and failed-payment nudges with your own copy, links and branding. |
| **Appointment & booking reminders** | Schedule reminders 24 hours and 1 hour before an appointment; daily, weekly and monthly recurrence is built in. |
| **Customer support alerts** | Receive every incoming reply on a signed webhook, create a ticket in your helpdesk and answer from your own CRM. |
| **Logistics & field updates** | Share a location pin with a driver, send delivery-proof photos, or tell a customer their technician is on the way. |
| **Lead alerts for sales** | When a form is submitted, send a formatted contact card to the sales number that needs to act on it. |
| **Internal operations** | Shift reminders, stock alerts and approval requests to staff. Use separate phones (instances) for separate teams. |

### Capabilities included on every plan

- Text, images, video, audio and voice notes
- PDFs, Office files, ZIP archives and uploads up to 25 MB
- Stickers, map locations and contact cards
- Signed incoming-message webhooks, plus delivery and read receipts
- Scheduled and recurring messages
- WhatsApp number verification before you send
- Multiple WhatsApp numbers per account, each with its own API key
- Human-typing simulation and automatic pause when you reply from your phone

> **Good rule of thumb:** if a customer would expect the update by SMS or email,
> they will appreciate it on WhatsApp. Message people who have opted in, and
> honour STOP requests — PingWapi tracks opt-outs for you.

---

## Quick Start

This walkthrough is for a first-time user. You do **not** need to create an
instance with `curl` before you have an account: connect your phone in the
dashboard, copy the credentials, then make API calls.

### Step 1 — Create your account and connect a phone

1. Open [https://pingwapi.com/signup](https://pingwapi.com/signup).
2. Enter your email, WhatsApp phone and password, then submit the 6-digit
   verification code sent to your email/WhatsApp.
3. In the dashboard, click **Connect Phone** and scan the QR code from
   **WhatsApp → Linked Devices → Link a Device** (or request an 8-digit
   pairing code).

Your phone is now an **instance**: one WhatsApp number your code can send from.

### Step 2 — Copy your credentials

Open **API Keys** in the dashboard and copy:

- `instanceId` — the UUID for the connected phone
- `apiKey` — the per-instance key beginning with `pw_`

The API key is shown once. Store it as a secret. It can send messages for that
phone, but it cannot create/delete phones, read your profile or manage billing.

### Step 3 — Send your first message

Once the dashboard shows the phone as **Connected**:

```bash
curl -X POST "https://api.pingwapi.com/send-message" \
  -H "Authorization: Bearer pw_YOUR_PHONE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "recipients": ["254712345678"],
    "type": "text",
    "text": "Hello from the PingWapi API! 👋",
    "simulateTyping": true
  }'
```

```json
{
  "success": true,
  "message": "Message sent successfully",
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "logId": "…",
  "recipient": "254712345678",
  "whatsappMessageId": "BAE5…",
  "typingDurationMs": 2240
}
```

### Optional — create an instance programmatically

Account-level endpoints use a **session JWT** from `POST /login`, not the
`pw_...` instance key. Use this only when onboarding phones outside the
dashboard.

```bash
# 1. Sign in as the account owner
curl -X POST "https://api.pingwapi.com/login" \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com", "password": "your-password"}'
# response includes: "access": "eyJ..." (valid 6 hours)

# 2. Create the phone with that session token
curl -X POST "https://api.pingwapi.com/instances" \
  -H "Authorization: Bearer eyJ_YOUR_SESSION_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Sales Line"}'
```

The create response returns the new `instanceId`, its `apiKey` and its
`webhookSecret`. Keep the `apiKey`; use it for messaging endpoints from then on.

---

## Authentication

PingWapi uses two credentials. Use the right one for the endpoint:

| Credential | How you get it | Use it for |
|---|---|---|
| **Instance API key** (`pw_...`) | Dashboard → API Keys, or the `apiKey` returned when an instance is created | Sending messages, checking numbers, QR/status, webhooks, schedules, media, message history |
| **Session JWT** (`eyJ...`) | `POST /login` with your account email/password | Account-level endpoints: list/create/rename/delete instances, rotate keys, profile, billing |

```
# Messaging endpoints
Authorization: Bearer pw_YOUR_PHONE_API_KEY
Content-Type: application/json

# Account-level endpoints
Authorization: Bearer <session access token from POST /login>
```

- Instance keys are **per phone**, issued once (and again on rotation).
- Keys are stored as SHA-256 hashes server-side — a lost key cannot be recovered; rotate it from the dashboard.
- An instance key can never read or control another account's phones, profile or billing.
- Session JWTs expire after 6 hours. The dashboard refreshes them silently; direct API clients should sign in again.
- Requests without valid credentials receive `401`; an instance key used on an account-level route is rejected.

---

## Instances & Phones

An **instance** is one WhatsApp number under your account. Create, list and
rotate-key need a session JWT from `POST /login`. QR, pairing, status, reconnect,
rename and delete accept the instance API key for that phone (or the session
JWT).

### List instances

`GET /instances` — every non-deleted instance with status and webhook URL.

### Create instance

`POST /instances`

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | no | Friendly label (defaults to `WhatsApp Phone #N`) |
| `webhook_url` | string | no | Initial incoming-message webhook URL |

### Get QR code

`GET /instances/{id}/qr`

Returns the QR payload and a base64 PNG. Re-triggers engine init when the session is disconnected. Status flow: `initializing` → `awaiting_scan` (with `qr` / `qrRaw`) → `connected`.

### Get QR image

`GET /instances/{id}/qr-image` — serves the QR as `image/png` for embedding in your own UI.

### Request pairing code

`POST /instances/{id}/pairing-code`

| Field | Type | Required |
|---|---|---|
| `phone_number` | string | yes |

### Check connection status

`GET /instances/{id}/status` — live status (`initializing` / `awaiting_scan` / `connected` / `disconnected`) plus health details.

### Reconnect / refresh QR

`POST /instances/{id}/reconnect` — restarts the session and issues a fresh QR without losing credentials.

### Reset session (change phone)

`POST /instances/{id}/reset-session` (aliases: `/reset_session`, `/relink`) — logs out the current phone so a different number can be linked.

### Rotate API key

`POST /instances/{id}/rotate-key` — issues a fresh key; the previous key is invalidated immediately. The phone stays connected.

### Rename instance

`PATCH /instances/{id}` — `{ "name": "Support Line", "device_imei_note": "Pixel 8" }`

### Delete instance

`DELETE /instances/{id}` — disconnects the session and soft-deletes the instance together with its API keys.

---

## Sending Messages

`POST /send-message` — one endpoint, eight payloads. Set `type` to `text`, `image`, `video`, `audio`, `document`, `sticker`, `location` or `contacts`.

Media is accepted as:
1. a public **`url`** (https, SSRF-protected, redirects refused),
2. **`base64`** content, or
3. a local media path from the upload endpoint (`/media/uploads/…`).

Common fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `instanceId` | uuid | yes | Instance to send from |
| `recipients` | string[] | yes | One or more numbers (E.164 or local — local numbers use your default country code) |
| `type` | string | yes | `text` · `image` · `video` · `audio` · `document` · `sticker` · `location` · `contacts` |
| `simulateTyping` | boolean | no | Human "typing…" indicator first (default `true`) |
| `enqueue` | boolean | no | Force async queue delivery (default: `true` for multi-recipient) |

**Delivery model:** a single recipient returns the WhatsApp message ID synchronously. Multiple recipients queue through Celery with safe pacing and return `202 Accepted` with `taskId` and per-recipient `messageIds`.

### Text

```bash
curl -X POST "https://api.pingwapi.com/send-message" \
  -H "Authorization: Bearer pw_YOUR_PHONE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "recipients": ["254712345678"],
    "type": "text",
    "text": "Hello! Your order #1042 has been dispatched."
  }'
```

### Image

JPEG / PNG / WebP / GIF, optional `caption`.

```json
{
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "recipients": ["254712345678"],
  "type": "image",
  "url": "https://example.com/receipt.png",
  "caption": "Your receipt 🧾"
}
```

### Video

MP4 / WebM / MOV up to 25 MB, optional `caption`.

```json
{
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "recipients": ["254712345678"],
  "type": "video",
  "url": "https://example.com/demo.mp4",
  "caption": "Watch how it works"
}
```

### Audio (and voice notes)

MP3 / M4A / OGG / WAV / AAC / OPUS. Set `ptt: true` to deliver as a push-to-talk **voice note**.

```json
{
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "recipients": ["254712345678"],
  "type": "audio",
  "url": "https://example.com/voice-note.m4a",
  "ptt": true,
  "mimetype": "audio/mp4"
}
```

### Document

PDF, DOC/DOCX, XLS/XLSX, PPT/PPTX, TXT, CSV, ZIP and more. `fileName` and `mimetype` are honoured.

```json
{
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "recipients": ["254712345678"],
  "type": "document",
  "url": "https://example.com/invoice_1042.pdf",
  "fileName": "Invoice_1042.pdf",
  "caption": "Please find your receipt attached."
}
```

### Sticker

WebP stickers via URL or base64. Stickers do not carry captions.

```json
{
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "recipients": ["254712345678"],
  "type": "sticker",
  "url": "https://example.com/thumbs-up.webp"
}
```

### Location

A map pin with optional name and address.

```json
{
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "recipients": ["254712345678"],
  "type": "location",
  "latitude": -1.286389,
  "longitude": 36.817223,
  "locationName": "PingWapi HQ",
  "locationAddress": "Nairobi, Kenya"
}
```

### Contact cards (vCard)

One or more contact cards — full Baileys contact objects or a raw vCard string.

```json
{
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "recipients": ["254712345678"],
  "type": "contacts",
  "text": "Support Team",
  "contacts": [
    {
      "displayName": "Mary Wanjiku",
      "vcard": "BEGIN:VCARD\nVERSION:3.0\nFN:Mary Wanjiku\nTEL;type=CELL:+254712345678\nEND:VCARD"
    }
  ]
}
```

---

## Check Number

`POST /check-number` — verify a number is registered on WhatsApp before you spend a send on it.

```bash
curl -X POST "https://api.pingwapi.com/check-number" \
  -H "Authorization: Bearer pw_YOUR_PHONE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890", "number": "254712345678"}'
```

```json
{
  "success": true,
  "exists": true,
  "number": "254712345678",
  "jid": "254712345678@s.whatsapp.net",
  "formatValid": true,
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

---

## Scheduled Messages

### Schedule

`POST /schedule-message` — all message types are schedulable.

| Field | Type | Required | Description |
|---|---|---|---|
| `instanceId` | uuid | yes | Instance to send from |
| `recipients` | string[] | yes | Recipient numbers |
| `type` | string | yes | Same types as `/send-message` |
| `scheduledAt` | ISO-8601 | yes | Future timestamp, e.g. `2026-12-01T09:00:00Z` |
| `recurrence` | string | no | `none` (default) · `daily` · `weekly` · `monthly` |
| `text` | string | no | Body (or caption for media) |
| `url` / `base64` | string | no | Media source for media types |
| `latitude` / `longitude` | number | no | Required when `type` is `location` |
| `contacts` | object[] \| string | no | Required when `type` is `contacts` |

```bash
curl -X POST "https://api.pingwapi.com/schedule-message" \
  -H "Authorization: Bearer pw_YOUR_PHONE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "recipients": ["254712345678"],
    "type": "text",
    "text": "Reminder: your subscription renews tomorrow.",
    "scheduledAt": "2026-12-01T09:00:00Z",
    "recurrence": "monthly"
  }'
```

### List

`GET /scheduled-messages?instanceId={id}` — up to 100 schedules. Filter with `?status=active|done|failed`.

### Cancel

`DELETE /scheduled-messages/{id}` — soft-deletes the schedule.

---

## Webhooks

When a customer messages your connected phone, PingWapi POSTs the event to your webhook URL as a signed envelope.

### Configure

`POST /instances/{id}/webhook`

| Field | Type | Description |
|---|---|---|
| `webhook_url` | string | HTTPS URL that receives events |
| `webhook_enabled` | boolean | Toggle delivery on/off |
| `human_override_minutes` | integer | Minutes (0–1440) automated sends pause for a chat after you reply from the handset (default 15) |

### Events

| Event | When |
|---|---|
| `messages.upsert` | A customer sends you a WhatsApp message |
| `message.ack` | One of your outbound messages is delivered (`status: "delivered"`) or read (`status: "read"`) |

### Incoming message envelope

```json
{
  "event": "messages.upsert",
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": 1764086400,
  "data": {
    "messageId": "BAE5…",
    "from": "254712345678",
    "remoteJid": "254712345678@s.whatsapp.net",
    "pushName": "Mary",
    "messageType": "text",
    "text": "Where is my order?",
    "timestamp": 1764086399
  }
}
```

### Delivery / read receipt envelope

```json
{
  "event": "message.ack",
  "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "timestamp": 1764086460,
  "data": {
    "messageId": "BAE5…",
    "status": "delivered",
    "ack": 3
  }
}
```

`status` is `"delivered"` for ack 3 and `"read"` for ack 4.

### Headers

| Header | Value |
|---|---|
| `X-PingWapi-Event` | `messages.upsert` |
| `X-PingWapi-Instance` | your instance ID |
| `X-PingWapi-Signature` | `sha256=<hex HMAC>` |

### Verifying the signature

The signature is an **HMAC-SHA256 of the raw request body**, keyed with your instance's `webhookSecret` (returned at instance creation). Always verify with a constant-time comparison.

**Python**

```python
import hashlib
import hmac
import os

SECRET = os.environ["PINGWAPI_WEBHOOK_SECRET"]  # instance webhookSecret

def verify_pingwapi_webhook(raw_body: bytes, signature_header: str) -> bool:
    expected = hmac.new(SECRET.encode(), raw_body, hashlib.sha256).hexdigest()
    received = signature_header.removeprefix("sha256=")
    return hmac.compare_digest(expected, received)
```

**Node.js**

```javascript
const crypto = require("crypto");

const SECRET = process.env.PINGWAPI_WEBHOOK_SECRET; // instance webhookSecret

function verifyPingWapiWebhook(rawBody, signatureHeader) {
  const expected = crypto.createHmac("sha256", SECRET).update(rawBody).digest("hex");
  const received = signatureHeader.replace(/^sha256=/, "");
  return crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(received));
}
```

### Delivery & retries

- Return `2xx` to acknowledge; anything else is treated as a failure.
- Retries: up to **5 attempts** with exponential backoff (10s → 20s → 40s → 80s → 160s).
- The full delivery history (status, attempts, response) is available from `GET /instances/{id}/webhook`.
- Replies containing STOP-style opt-out words automatically suppress future campaigns to that contact.

---

## Message History

`GET /instances/{id}/messages?limit=50` — newest-first log of sent and received messages.

| Field | Description |
|---|---|
| `limit` | 1–300 rows (default capped at 150) |
| `status` | `queued` · `sent` · `failed` · `received` · `paused_human_override` |

---

## Media Upload

Upload files first, then send the returned `mediaUrl` path in `/send-message`'s `url` field — ideal for large files. Files are stored on the platform volume and read locally by the engine (no public exposure). The response also includes the base64 payload for immediate use.

```bash
curl -X POST "https://api.pingwapi.com/media/upload" \
  -H "Authorization: Bearer pw_YOUR_PHONE_API_KEY" \
  -F "file=@./invoice_1042.pdf"
```

```json
{
  "success": true,
  "fileName": "invoice_1042.pdf",
  "mimetype": "application/pdf",
  "inferredType": "document",
  "mediaUrl": "/media/uploads/3f9c2a8d41ab_invoice_1042.pdf",
  "base64": "JVBERi0xLjQK…"
}
```

Supported formats:

| Category | Formats |
|---|---|
| Images | PNG, JPEG, GIF, WebP |
| Documents | PDF, DOC/DOCX, XLS/XLSX, PPT/PPTX, TXT, CSV, ZIP |
| Audio | MP3, M4A, OGG, WAV, AAC, OPUS |
| Video | MP4, WebM, MOV, AVI, 3GP |

- Max file size: **25 MB**.
- File contents are magic-byte validated against the declared type — a renamed file cannot sneak through.
- External media URLs are resolved with SSRF protection: private/link-local addresses are rejected and redirects are refused.

---

## Rate Limits & Quotas

PingWapi never charges per message, but it paces sends to keep your number healthy and out of WhatsApp's spam filters. Requests beyond the per-minute limit receive `429`.

| Plan | Phones | Requests / minute | Monthly messages |
|---|---|---|---|
| Basic | 1 | 60 | Unlimited |
| Pro | 5 | 120 | Unlimited |
| Business | 10 | 300 | Unlimited |

---

## Error Codes

Every error returns a JSON body with `success: false` and a human-readable `message`.

| Status | Meaning |
|---|---|
| `400` | Malformed payload — missing required field, bad media URL, invalid recurrence, or past `scheduledAt` |
| `401` | Missing or invalid API key |
| `403` | Plan limit reached (phones, quota) or subscription expired |
| `404` | Instance not found or not owned by this key |
| `409` | Human Override Auto-Pause is active for that chat |
| `425` | Instance not connected yet — scan the QR first (Too Early) |
| `429` | Per-minute rate limit exceeded |
| `503` | Messaging engine or queue temporarily unreachable |

---

## Code Samples by Language

### Send a text message — Python

```python
import requests

url = "https://api.pingwapi.com/send-message"
headers = {
    "Authorization": "Bearer pw_YOUR_PHONE_API_KEY",
    "Content-Type": "application/json",
}
payload = {
    "instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "recipients": ["254712345678"],
    "type": "text",
    "text": "Hello from Python!",
}

response = requests.post(url, json=payload, headers=headers, timeout=30)
print(response.status_code, response.json())
```

### Send a text message — Node.js

```javascript
const response = await fetch("https://api.pingwapi.com/send-message", {
  method: "POST",
  headers: {
    Authorization: "Bearer pw_YOUR_PHONE_API_KEY",
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    instanceId: "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    recipients: ["254712345678"],
    type: "text",
    text: "Hello from Node!",
  }),
});

const data = await response.json();
console.log(data);
```

### Send a text message — PHP

```php
<?php
$ch = curl_init("https://api.pingwapi.com/send-message");
$payload = json_encode([
    "instanceId" => "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "recipients" => ["254712345678"],
    "type" => "text",
    "text" => "Hello from PHP!",
]);

curl_setopt($ch, CURLOPT_HTTPHEADER, [
    "Authorization: Bearer pw_YOUR_PHONE_API_KEY",
    "Content-Type: application/json",
]);
curl_setopt($ch, CURLOPT_POST, true);
curl_setopt($ch, CURLOPT_POSTFIELDS, $payload);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

$response = curl_exec($ch);
curl_close($ch);
echo $response;
```

### Send a text message — Go

```go
package main

import (
	"bytes"
	"fmt"
	"net/http"
)

func main() {
	payload := []byte(`{
		"instanceId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
		"recipients": ["254712345678"],
		"type": "text",
		"text": "Hello from Go!"
	}`)
	req, _ := http.NewRequest("POST", "https://api.pingwapi.com/send-message", bytes.NewBuffer(payload))
	req.Header.Set("Authorization", "Bearer pw_YOUR_PHONE_API_KEY")
	req.Header.Set("Content-Type", "application/json")

	resp, err := (&http.Client{}).Do(req)
	if err == nil {
		defer resp.Body.Close()
		fmt.Println("Status:", resp.Status)
	}
}
```

### Send a text message — C#

```csharp
using System.Net.Http;
using System.Net.Http.Headers;
using System.Text;

using var client = new HttpClient();
client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", "pw_YOUR_PHONE_API_KEY");

var json = "{\"instanceId\":\"a1b2c3d4-e5f6-7890-abcd-ef1234567890\",\"recipients\":[\"254712345678\"],\"type\":\"text\",\"text\":\"Hello from C#!\"}";
var content = new StringContent(json, Encoding.UTF8, "application/json");
var response = await client.PostAsync("https://api.pingwapi.com/send-message", content);
Console.WriteLine(await response.Content.ReadAsStringAsync());
```

---

## Support

- Live chat: the chat bubble on every page of the dashboard and website.
- Email: `support@pingwapi.com`

---

*WhatsApp™ and Meta™ are registered trademarks of Meta Platforms, Inc. PingWapi is an independent software platform and is not affiliated with, endorsed, or sponsored by Meta Platforms, Inc.*
