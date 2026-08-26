# Booking Page Design (v1)

- Status: Draft for review
- Date: 2026-08-26
- Repo: greenzofreaks (static site on Vercel)
- Related: tip page (`/tip`), home page dispatch list

## Overview

A booking/inquiry page at `/booking` where a potential client picks a
date and describes their event, then submits. The request is emailed to
`greenzofreaks@gmail.com`. The DJ reviews and confirms manually. No
availability or calendar is exposed publicly in v1.

This is Approach B from brainstorming: the form posts to a Vercel
serverless function on our own domain, which sends the email via Resend.
This keeps everything on-brand and on `greenzofreaks.com`, and it is the
foundation for the later private admin page (v3).

## Goals

- Let someone submit a gig request in one short, on-brand form.
- Deliver each request to `greenzofreaks@gmail.com` reliably, with the
  requester's email as reply-to so the DJ can just hit reply.
- Keep the DJ's calendar and availability private.
- Build on a foundation that grows into a private admin page and,
  eventually, an opt-in public calendar, without a rewrite.

## Non-goals (v1, YAGNI)

- No public calendar or availability display.
- No self-serve confirmed booking (the DJ approves manually).
- No deposit or payment at booking time (Stripe stays on `/tip` for now).
- No account, login, or stored bookings yet (that arrives in v3).
- No contracts, invoices, or automated scheduling.

## User flow

1. Visitor lands on `/booking` (from the home page "Book the DJ"
   dispatch entry or a direct link).
2. Fills the form, submits.
3. Client validates required fields inline. On success it POSTs JSON to
   `/api/book`.
4. The function validates, checks the spam honeypot, and emails the
   request via Resend.
5. Client shows an on-brand "TRANSMISSION RECEIVED" success state
   without a page reload. On failure it shows an error plus a `mailto:`
   fallback so the lead is never lost.

## Architecture

- Static page: `booking/index.html`, served at `/booking` by Vercel,
  styled in the existing field-log system (near-black `#04080a`,
  phosphor green accents, monospace, sharp corners, scanline/grain FX).
- Serverless function: `api/book.js` (Node runtime on Vercel). Accepts
  POST JSON, validates, sends email. Because Vercel auto-detects the
  `api/` directory, the otherwise-static site gains this one endpoint
  with no framework change.
- Email delivery: Resend HTTP API (`POST https://api.resend.com/emails`).
- Home page: add a "Book the DJ" dispatch entry alongside the tip jar.

## Form specification

Required fields (client and server enforce):

- Event date, `<input type="date">`.
- Event type, `<select>`: Wedding, Club / Bar, Private Party, Corporate,
  Festival, Other.
- Venue or city, text.
- Name, text.
- Email, `<input type="email">`, format validated.
- Tell me about the gig, textarea.

Optional fields:

- Start time and end time, `<input type="time">`.
- Expected headcount, `<input type="number" min="1">`.
- Budget range, `<select>`: Under $300, $300–$600, $600–$1000, $1000+,
  Not sure yet.
- Phone, `<input type="tel">`.

Hidden field:

- Honeypot, a visually hidden text input (for example named `company`).
  Real users leave it empty. If filled, the server treats the submission
  as spam.

## Backend: `api/book.js`

Request: `POST /api/book`, `Content-Type: application/json`, body holds
the form fields plus the honeypot.

Behavior:

1. Reject non-POST with 405.
2. Parse and validate. Required fields present, email matches a basic
   pattern, event date parses. On failure return 400 with a short
   `{ ok: false, error }`.
3. If the honeypot is non-empty, return 200 `{ ok: true }` without
   sending anything (silently drop, so bots see success).
4. Build the email and send via Resend. On Resend failure return 500
   `{ ok: false, error: "send_failed" }`.
5. On success return 200 `{ ok: true }`.

Validation and honeypot logic live in a small pure module
(`api/_lib/validate.js`) so they can be unit tested without network.

## Email

- Provider: Resend.
- From: `FROM_EMAIL` (for example `bookings@greenzofreaks.com`), using a
  verified sender domain.
- To: `TO_EMAIL` (`greenzofreaks@gmail.com`).
- Reply-To: the requester's email, so replying reaches the client.
- Subject: `New gig request: {date} · {event type} · {city}`.
- Body: plain text plus a simple HTML version listing every submitted
  field in a readable order. Missing optional fields are omitted.

## Configuration and secrets

Vercel environment variables (Production and Preview):

- `RESEND_API_KEY` (secret).
- `TO_EMAIL` = `greenzofreaks@gmail.com`.
- `FROM_EMAIL` = `bookings@greenzofreaks.com`.

No secrets live in the repo. The client never sees these; only the
serverless function reads them.

## Security and spam

- Honeypot is the primary spam defense in v1.
- The function only accepts POST and only reads the fields it expects.
- Optional, best effort: a light per-request throttle. Serverless memory
  is not shared across invocations, so any in-memory limit is advisory
  only. Real rate limiting is deferred to v3 if abuse appears.
- No user input is reflected back to the browser, avoiding injection in
  the success path. The email body escapes HTML for the HTML version.

## Error handling and fallback

- Inline field validation with clear messages before submit.
- Network or server error shows an error state and a
  `mailto:greenzofreaks@gmail.com` link prefilled with a subject, so the
  visitor can still reach the DJ if the API is down.
- Success and error states match the field-log aesthetic.

## Accessibility and style

- Labels tied to inputs, keyboard friendly, visible focus states.
- Respect `prefers-reduced-motion` (as the rest of the site does).
- Reuse the existing CSS design system so the page feels native.

## Testing strategy

- TDD the pure validation and honeypot module: required-field checks,
  email format, honeypot detection, date parsing.
- Local integration with `vercel dev`, posting sample payloads.
- One real end-to-end submission after Resend is configured, confirming
  the email arrives at `greenzofreaks@gmail.com` with correct reply-to.
- Manual form pass on mobile widths.

## Manual setup steps (at build time)

1. Create a Resend account.
2. Add and verify `greenzofreaks.com` in Resend. This adds a few DNS
   records at Cloudflare (SPF, DKIM, and a return-path), separate from
   the site's A/CNAME records. Exact records come from Resend.
3. Create a Resend API key.
4. Add `RESEND_API_KEY`, `TO_EMAIL`, `FROM_EMAIL` in Vercel project
   settings for Production and Preview.

## v3 foundation (future, not built now)

- The same `/api/book` endpoint also inserts each booking into Supabase
  (already used by the plant app), so requests are stored, not just
  emailed.
- A private, password-protected admin page on the site lists and manages
  requests.
- An opt-in public calendar can then be layered on when the DJ chooses to
  expose availability.

## Out of scope

Payments at booking, contracts, automated availability, public calendar,
and stored data all wait for later phases.
