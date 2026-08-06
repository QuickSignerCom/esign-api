# eSign API: A Developer's Guide to Integrating Electronic Signatures (REST, PAdES, ESIGN & EIDAS Compliant)

> Written by the QuickSigner team. A practical, example-driven guide to integrating the QuickSigner REST API — upload documents, request signatures, track status, and download legally binding signed PDFs from your own application.

The **QuickSigner eSign API** is a public REST API that lets you embed legally binding electronic signing directly into your own applications — CRMs, onboarding flows, contract tools, or anything that needs documents signed. Signatures are **PAdES** (PDF Advanced Electronic Signatures), valid and verifiable over time under EU **eIDAS**, US **ESIGN**, and **UETA** law. The platform is **ISO/IEC 27001:2022 certified**, GDPR compliant, and pricing is usage-based at roughly **$0.30 per envelope**.

- **Base URL:** `https://app.quicksigner.com/api/v1`
- **Full interactive reference:** https://quicksigner.stoplight.io/

---

## What Is an eSignature API?

An eSignature API is a programmatic interface for the whole signing workflow: your application sends a document, defines who signs it and where, and receives back a completed, tamper-evident PDF plus an audit trail — without the signer ever leaving your product and without you writing a line of cryptography.

Concretely, the API takes over four things you would otherwise have to build:

| You'd otherwise build | The API does it |
|---|---|
| Signer identity, secure per-signer links, reminders | One `recipients[]` array; each signer gets a tokenised `signUrl` |
| Field placement, validation, signing UI | `fields[]` with `x`/`y`/`pageNum`; hosted or embedded signing view |
| Cryptographic sealing, certificate chain, timestamping | PAdES seal with an AATL certificate + qualified timestamp, applied server-side |
| Audit trail, evidence for disputes | A signing certificate per completed request |

The minimum viable integration is two HTTP calls:

```
POST /files      →  fileName
POST /requests   →  signUrl per recipient + a webhook when it's done
```

Everything else in this guide — templates, signing order, embedded signing, tag extraction — is optional on top of those two.

**Where an eSignature API fits:** CRM "send contract" buttons, SaaS onboarding and KYC flows, HR offer letters, vendor and NDA workflows, invoice approvals, and any product where a human has to agree to something before the next step can run.

---

## How It Works

The core integration is two calls:

1. **Upload a document** → `POST /files` returns a `fileName`.
2. **Create a sign request** → `POST /requests` references that `fileName`, defines recipients, and notifies them to sign.

From there you can track status, download the signed PDF and signing certificate, send reminders, use templates, and subscribe to webhooks for completion events.

---

## Table of Contents

- [What Is an eSignature API?](#what-is-an-esignature-api)
- [Authentication](#authentication)
- [1. Upload a Document](#1-upload-a-document)
- [2. Create a Sign Request](#2-create-a-sign-request)
- [Embedded Signing](#embedded-signing--keep-signers-in-your-app)
- [3. List & Retrieve Sign Requests](#3-list--retrieve-sign-requests)
- [4. Download the Signed Document & Certificate](#4-download-the-signed-document--certificate)
- [5. Reminders, Cancel & Programmatic Signing](#5-reminders-cancel--programmatic-signing)
- [Templates](#templates)
- [Fields & Recipient Options](#fields--recipient-options)
- [Webhooks](#webhooks)
- [Signing Order & Multiple Recipients](#signing-order--multiple-recipients)
- [AATL, PAdES & Long-Term Validity](#aatl-pades-and-long-term-validity)
- [Compliance & Security](#compliance--security)
- [Choosing an eSignature API](#choosing-an-esignature-api-what-matters-in-production)
- [FAQ](#faq)

---

## Authentication

Every request is authenticated with your account **API key**, sent in the **`Authorization`** header.

```
Authorization: YOUR_API_KEY
```

**Getting a key:**

- **Free development account** — for testing. Sign up for a free API development account, then enable the API on your *My Account* page. You can create unlimited sign requests, but completed documents are **watermarked**.
- **Production** — sign up, upgrade to the **PRO plan**, then enable the API on your *My Account* page.

Once enabled, your API key is shown on the **API page** in your account.

---

## 1. Upload a Document

Upload the PDF/DOC/DOCX you want signed. **Maximum file size: 15 MB.** The response returns a unique `fileName` you'll use in the next call.

```bash
curl --request POST \
  --url https://app.quicksigner.com/api/v1/files \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Accept: application/json' \
  --header 'Content-Type: multipart/form-data' \
  --form file=@contract.pdf
```

**Response `200`**

```json
{
  "fileName": "a1b2c3d4-...-contract.pdf"
}
```

> Note: the Create Sign Request step requires a **.pdf** file (`fileName` must reference a PDF).

---

## 2. Create a Sign Request

Reference the `fileName` from step 1, add one or more recipients, and QuickSigner emails each recipient a secure signing link to begin the signing process.

```bash
curl --request POST \
  --url https://app.quicksigner.com/api/v1/requests \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Accept: application/json' \
  --header 'Content-Type: application/json' \
  --data '{
    "fileName": "a1b2c3d4-...-contract.pdf",
    "recipients": [
      {
        "type": "EMAIL",
        "email": "jane@example.com",
        "name": "Jane Doe",
        "fields": [
          {
            "type": "SIGNATURE",
            "pageNum": 1,
            "x": 100, "y": 600, "width": 180, "height": 60,
            "required": true
          }
        ],
        "options": ["one_click_sign"]
      }
    ],
    "signInOrder": false,
    "senderName": "Acme Corp",
    "emailSubject": "Please sign your contract"
  }'
```

**Key request fields**

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `fileName` | string | yes | From the upload step; must be a `.pdf` (min 4 chars) |
| `recipients` | array | yes | One or more recipient objects |
| `recipients[].type` | string | yes | `EMAIL` (recipient is notified by email) |
| `recipients[].email` | string | for EMAIL | Recipient email |
| `recipients[].name` | string | yes | Recipient display name |
| `recipients[].fields` | array | no | Signature/other fields to place — see [Fields](#fields--recipient-options) |
| `recipients[].options` | array | no | `one_click_sign`, `disable_notifs`, `is_copy_only` |
| `recipients[].signedRedirectUrl` | string | no | Redirect after signing (signature applied async — see webhooks) |
| `signInOrder` | boolean | no | Enforce sequential signing |
| `extractTags` | boolean | no | Auto-generate fields from invisible text tags in the document (default `false`) |
| `senderName` | string | no | Shown to signer under "Sent by:" and in the notification email |
| `emailSubject` | string | no | Custom email subject (≤255 chars) |
| `emailMessage` | string | no | Inserted into the default email body (≤10000 chars) |
| `hideInfoPanel` | boolean | no | Hide left info panel (useful for iframe embedding) |
| `templateId` | string | no | Use a saved template instead of uploading a file |

**Response `200`** (abbreviated)

```json
{
  "id": "req_...",
  "createdAt": "2026-01-01T12:00:00Z",
  "fileName": "a1b2c3d4-...-contract.pdf",
  "status": "IN_PROGRESS",
  "recipients": [
    {
      "id": "rcp_...",
      "type": "EMAIL",
      "email": "jane@example.com",
      "name": "Jane Doe",
      "signUrl": "https://app.quicksigner.com/sign/...",
      "phoneUrl": "..."
    }
  ]
}
```

The `signUrl` is the unique link each recipient uses to sign. (Add `"include": ["phone_urls"]` to also get a shorter SMS-friendly URL for phone recipients.)

---

## Embedded Signing — Keep Signers in Your App

The `signUrl` returned for each recipient can be loaded in an iframe inside your own interface, so signers never see a redirect, a new tab, or a branded interstitial. The output PDF is identical — same PAdES seal, same AATL certificate.

Three parameters do the work:

| Parameter | Where | Effect |
|---|---|---|
| `hideInfoPanel` | request | Removes the left info panel so the signing view fits your layout |
| `recipients[].signedRedirectUrl` | recipient | Where the signer lands after signing (your own thank-you view) |
| `disable_notifs` | `recipients[].options` | Suppresses email for a signer who is already looking at your screen |

```bash
curl --request POST \
  --url https://app.quicksigner.com/api/v1/requests \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "fileName": "a1b2c3d4-...-contract.pdf",
    "hideInfoPanel": true,
    "recipients": [
      {
        "type": "EMAIL",
        "email": "jane@example.com",
        "name": "Jane Doe",
        "options": ["disable_notifs", "one_click_sign"],
        "signedRedirectUrl": "https://yourapp.com/contracts/42/signed",
        "fields": [
          { "type": "SIGNATURE", "pageNum": 1,
            "x": 100, "y": 600, "width": 180, "height": 60, "required": true }
        ]
      }
    ]
  }'
```

Then embed the returned `signUrl`:

```html
<iframe
  src="https://app.quicksigner.com/sign/..."
  width="100%"
  height="900"
  style="border:0"
  allow="camera; clipboard-write">
</iframe>
```

> **Important:** `signedRedirectUrl` can fire *before* the signature is written into the PDF — sealing is asynchronous. Use it for the UI transition only. Treat the `REQUEST_COMPLETED` webhook as the single source of truth before you download, display, or archive the signed file.

---

## 3. List & Retrieve Sign Requests

**List** (paginated via `cursor` + `limit`, default 10):

```bash
curl --request GET \
  --url 'https://app.quicksigner.com/api/v1/requests?limit=10' \
  --header 'Accept: application/json'
```

```json
[
  { "id": "req_...", "createdAt": "...", "docName": "contract.pdf",
    "status": "IN_PROGRESS", "completedAt": null }
]
```

Status values: `IN_PROGRESS`, `COMPLETED`, `CANCELED`, `REJECTED`.

**Retrieve one** (full detail including per-recipient `sigStatus`, `order`, `fields`):

```bash
curl --request GET \
  --url https://app.quicksigner.com/api/v1/requests/{requestId} \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Accept: application/json'
```

---

## 4. Download the Signed Document & Certificate

QuickSigner returns short-lived signed URLs rather than raw bytes.

**Document being signed** (temporary, ~15 min):

```bash
curl --request GET \
  --url https://app.quicksigner.com/api/v1/requests/{requestId}/temp_download_url \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Accept: application/json'
# → { "requestId": "...", "docUrl": "https://..." }
```

**Completed signed PDF** (URL valid ~2 hours; returns `403` if the request isn't completed yet):

```bash
curl --request GET \
  --url https://app.quicksigner.com/api/v1/requests/{requestId}/completed_download_url \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Accept: application/json'
# → { "requestId": "...", "docUrl": "https://..." }
```

**Signing certificate** (URL valid ~15 min; `403` until completed):

```bash
curl --request GET \
  --url https://app.quicksigner.com/api/v1/requests/{requestId}/download_certificate_url \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Accept: application/json'
# → { "requestId": "...", "certificateUrl": "https://..." }
```

---

## 5. Reminders, Cancel & Programmatic Signing

**Remind a recipient** who hasn't signed (email or SMS depending on their type):

```bash
curl --request POST \
  --url https://app.quicksigner.com/api/v1/recipients/{recipientId}/remind \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Accept: application/json' --header 'Content-Type: application/json'
```

**Cancel** an in-progress request (only the sender, only while `IN_PROGRESS`):

```bash
curl --request POST \
  --url https://app.quicksigner.com/api/v1/requests/{requestId}/cancel \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Accept: application/json' --header 'Content-Type: application/json'
# → { "success": true, "message": "Request canceled" }
```

**Sign a request** programmatically — only when your API key's account is itself a recipient, has a default signature configured, and has signature fields assigned. Returns `204`; the signature is applied asynchronously (watch webhooks):

```bash
curl --request POST \
  --url https://app.quicksigner.com/api/v1/requests/{requestId}/sign \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Content-Type: application/json'
```

**Sign on behalf of a recipient** — for specific, consent-tracked cases (e.g. bulk signing with explicit recipient consent). Requires the recipient's sign-URL `token` (from the Create Request response) and a `signatureText`:

```bash
curl --request POST \
  --url https://app.quicksigner.com/api/v1/recipients/sign-on-behalf \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Accept: application/json' --header 'Content-Type: application/json' \
  --data '{ "token": "eyJhbGciOi...", "signatureText": "John Doe" }'
```

---

## Templates

List saved templates (reusable recipient/field layouts). Reference a returned `id` as `templateId` in Create Sign Request to skip uploading a file — fields from the template, request, and any extracted tags are merged.

```bash
curl --request GET \
  --url https://app.quicksigner.com/api/v1/templates \
  --header 'Authorization: YOUR_API_KEY' \
  --header 'Accept: application/json'
```

---

## Fields & Recipient Options

**Field types:** `SIGNATURE`, `STAMP`, `TEXT`, `CHECKBOX`, `FILE`, `DATE`, `TEXTAREA`.

A field is positioned on the document with `x`, `y`, `width`, `height`, `pageNum` (all required), plus optional `description`, `required`, `customId` (your internal id), and `fixed` (locks position so the signer can't move it). For checkboxes, `groupId` + `required` makes a group where at least one must be checked, and `radioMode` enables radio-group behavior.

**Recipient options:**

- `one_click_sign` — the recipient just clicks the pre-positioned signature field, auto-filled with their name.
- `disable_notifs` — suppress notifications for that recipient.
- `is_copy_only` — recipient receives a copy only (no signing).

**Auto-placing fields with tags:** set `extractTags: true` and embed tag markers in the document text (style them white to stay invisible) to generate fields automatically.

---

## Webhooks

Rather than polling, configure webhook events in the app dashboard → **API** page. The key events for completion handling are **`RECIPIENT_SIGNED`** and **`REQUEST_COMPLETED`**. Because signatures are applied **asynchronously**, always wait for these events before downloading or displaying a signed document — a `signedRedirectUrl` may fire before the signature is on the PDF.

---

## Signing Order & Multiple Recipients

Add multiple objects to `recipients[]`. Set `signInOrder: true` to enforce sequential signing (each recipient signs in the order listed); leave it `false` for parallel signing.

```json
{
  "fileName": "...pdf",
  "signInOrder": true,
  "recipients": [
    { "type": "EMAIL", "email": "a@example.com", "name": "A", "fields": [ /* ... */ ] },
    { "type": "EMAIL", "email": "b@example.com", "name": "B", "fields": [ /* ... */ ] }
  ]
}
```

---

## AATL, PAdES and Long-Term Validity

Three layers stack to make a signature hold up years later:

- **PAdES** (PDF Advanced Electronic Signatures) — the signature is embedded *inside* the PDF rather than attached alongside it, and any byte changed after signing is detectable.
- **AATL** (Adobe Approved Trust List) — the sealing certificate is on the list Adobe ships trust for, so Acrobat and Reader show "Signature is valid" with no warnings and no extra software on the verifier's machine.
- **Qualified timestamp** — a trusted time source gives long-term validity (LTV), so the signature stays verifiable after the signing certificate itself expires.

A practical way to test this: complete a sign request, download the PDF via `completed_download_url`, and open it in Adobe Acrobat Reader. The signature panel should show a valid signature with no trust warnings. If you are evaluating providers, run exactly this test on each one — it is the fastest way to tell a real certificate-based seal from a pasted signature image.

---

## Compliance & Security

- **ISO/IEC 27001:2022** certified; encrypted storage; PKI with cloud certificates.
- **eIDAS** (EU), **ESIGN Act** and **UETA** (US) compliant — signatures carry the same legal weight as handwritten ones for contracts, NDAs, HR and vendor agreements, and invoices.
- **GDPR** compliant. Documents are legally recognized across the US, UK, and EU.
- A signing **certificate** is available per completed request for audit evidence.

---

## Choosing an eSignature API: What Matters in Production

Most eSignature APIs look interchangeable on the pricing page. The differences surface later — during an audit, during a volume spike, or during a migration. Worth checking before you commit:

1. **Cryptographic standard.** PAdES with LTV timestamping means your PDFs still validate in five years. A rendered signature image with a separate audit log does not.
2. **Security and compliance posture.** ISO/IEC 27001 certification, encryption at rest, and where the signing keys actually live.
3. **Legal framework coverage.** eIDAS (EU/UK), ESIGN and UETA (US), GDPR for the data. Check the jurisdictions your signers are actually in.
4. **Pricing model.** Per-envelope ($/document) versus per-seat ($/user). Model both against your real volume curve — the crossover point is usually further out than vendors imply.
5. **Integration surface.** Templates, bulk send, webhooks, embedded signing, field tagging, attachments.
6. **Export and lock-in.** Can you pull every completed document *and* its audit trail out, in bulk, without a support ticket?
7. **Court-readiness.** Per-request signing certificate, IP and timestamp metadata, and a signer experience that demonstrates intent.

QuickSigner's positions on these: PAdES + AATL + qualified timestamp, ISO/IEC 27001:2022 certified, eIDAS/ESIGN/UETA/GDPR, per-document pricing at $0.30 on top of a Pro plan, and a downloadable signing certificate per completed request.

---

## FAQ

**Do signers need an account or a digital certificate?**
No. Recipients sign via a secure link using only an email address and internet access.

**What file types can I upload?**
PDF, DOC, and DOCX, up to 15 MB. The sign request itself references a PDF `fileName`.

**How do I authenticate?**
A single API key in the `Authorization` header — no OAuth flow, no token refresh. Enable the API on your *My Account* page; the key then appears on the **API** page.

**How do I know when a document is fully signed?**
Subscribe to the `REQUEST_COMPLETED` webhook (and `RECIPIENT_SIGNED` for per-signer events). Signatures apply asynchronously, so webhooks are more reliable than redirect timing.

**Does the API support webhooks?**
Yes. Register an endpoint in the dashboard → **API** page and QuickSigner POSTs events as documents are viewed, signed, completed, or expire. `REQUEST_COMPLETED` is the one to build completion logic on.

**Can I keep signers inside my own app?**
Yes — load the returned `signUrl` in an iframe and set `hideInfoPanel: true`. See [Embedded Signing](#embedded-signing--keep-signers-in-your-app).

**Will the signed PDF validate in Adobe Acrobat?**
Yes. The seal uses an AATL-listed certificate, so Acrobat and Reader show a valid signature without extra configuration.

**Can I test before paying?**
Yes. Sign up for a free **API development account** to build and test integrations — you can create unlimited sign requests, though completed documents are watermarked. Upgrade to the **PRO plan** for production (unwatermarked) use.

**What does it cost in production?**
A Pro plan plus $0.30 per signed document.

**Is the signature legally binding?**
Yes, for most business documents — PAdES-sealed with an AATL certificate, carrying legal weight under US ESIGN/UETA and EU/UK eIDAS. It is not a substitute for legal advice on document types your jurisdiction specifically excludes (wills, some property transfers, certain court filings).

**Where's the full, interactive reference?**
https://quicksigner.stoplight.io/ — including a "Try it" console for every endpoint.

---

*Affiliation disclosure: this guide is published by QuickSigner. Endpoints, fields, limits, and the authentication header above are drawn directly from the official QuickSigner REST API reference.*

---

## About QuickSigner

QuickSigner is a fast, secure, ISO 27001-certified Electronic Signature Software for collecting legally binding electronic signatures. Learn more at **[quicksigner.com](https://quicksigner.com)**.
