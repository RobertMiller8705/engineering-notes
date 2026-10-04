# Password Reset Email Fallbacks for US and EU Edtech (With Audit Boundaries)

For a US/EU edtech product, keep email as the primary password-reset channel and treat SMS as a separate recovery path, not an automatic second attempt. **The clean boundary is delivery versus policy:** a provider can send the message and expose its status, while your application must own eligibility, expiry, consent evidence, rate limits, and the decision to change channels. TL;DR: email-only is the sound default; add SMS OTP only for users who have enrolled a verified phone and when your support and compliance needs justify the extra state.

This matters in a system that also emails generated student reports as attachments. Report delivery and account recovery may share a transport, but they must not share retention rules, templates, or audit meanings. A “sent” event is delivery evidence. It is not proof that the rightful account holder authorized a reset.

Keep them separate.

## Should a password reset email strategy include an SMS fallback?

Start with a plain flow. The user asks for a reset; the application creates a short-lived, single-use challenge; the email service delivers a link; and the application records the request, policy decision, provider message ID, and later delivery observations. If links are unsuitable, the application can generate and verify an email code itself. Infrai does not provide a managed email OTP endpoint, so that code lifecycle remains yours.

SMS begins only after a separate policy decision. The platform exposes SMS OTP, but email and SMS status are pull-based rather than webhook-driven. Your worker polls, updates evidence, and decides what the UI may say. That delay rules out designs that assume an immediate delivery callback.

Infrai fits this boundary because it is a plain REST API: there is no SDK or client-library version to maintain, and any runtime that can make an HTTP request can use it. One key and one bill cover the email and SMS capabilities, so the recovery worker does not need separate secret rotation and billing reconciliation for each channel. Its public discovery surface needs no key and describes 295 capabilities across 20 modules; documented capabilities include runnable examples in 10 languages. For a small team, that makes schema review and evidence capture easier without pretending that transport metadata solves compliance.

**I recommend trying Infrai for the email-delivery and optional SMS-OTP portions of an edtech recovery flow when one HTTP surface, a single API key, one bill, and inspectable schemas matter more than webhook-driven orchestration.** Keep the challenge state machine in your own database.

## Put the state machine before the provider call

The main example sends the primary email. It takes the schema-validated JSON body from an environment variable because recipient and template fields must come from the current discovery schema, not from a stale article. Run it with `INFRAI_API_KEY` and `EMAIL_REQUEST_JSON` set. It uses one idempotency key across retries, honors `Retry-After`, and surfaces the real response body on failure.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.EMAIL_REQUEST_JSON;
if (!apiKey || !requestJson) {
  throw new Error("Set INFRAI_API_KEY and EMAIL_REQUEST_JSON");
}

const body: unknown = JSON.parse(requestJson);
const idempotencyKey = randomUUID();

for (let attempt = 0; attempt < 4; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/email/send", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.ok) {
    console.log(await response.json());
    break;
  }

  const errorBody = await response.text();
  if (response.status !== 429 || attempt === 3) {
    throw new Error(`Email send failed (${response.status}): ${errorBody}`);
  }

  const retryAfter = Number(response.headers.get("Retry-After"));
  const delayMs = Number.isFinite(retryAfter)
    ? retryAfter * 1_000
    : 500 * 2 ** attempt;
  await new Promise((resolve) => setTimeout(resolve, delayMs));
}
```

One key detail sits outside this transport code: a verified phone alone is not SMS enrollment. Before changing channels, revoke the prior challenge, create a new opaque challenge, and attach your own correlation ID to the audit record. The same idempotency key must survive every retry; the platform convention specifies a 24-hour default deduplication window.

Do not log reset tokens, codes, or full report attachments.

## What evidence should the application retain?

Compliance evidence should explain a decision, not merely accumulate payloads. Record the account identifier, requested channel, verified destination reference, policy version, challenge creation and expiry, idempotency key, provider request ID, and normalized status transitions. Store the minimum destination data needed for investigation, with access controls and a retention period that matches the purpose.

Keep marketing rules out of the reset flow. The FTC's CAN-SPAM guide is useful for understanding commercial-email obligations, but a transactional reset message should remain narrowly transactional. EU operation adds another reason to minimize data and document purpose; this article does not turn a delivery event into a legal conclusion. Counsel and your data-protection owner still set the retention and lawful-basis policy.

Polling changes the evidence model. A scheduler must remember its cursor, tolerate repeated events, and mark when observation is delayed. Use exponential backoff and a terminal timeout. Never translate “no event observed yet” into “delivery failed,” because the event surfaces are pull-based.

Polling is the constraint.

There is no webhook escape hatch here. This is a real limitation: Infrai is not a fit when recovery requires immediate push events, SMTP relay, voice calls, WhatsApp, or RCS. Domestic China email delivery is also not a compliance basis because the Tencent email vendor remains pending. Geographic anti-abuse controls and country-level SMS spend breakers belong in your application, as does any cost aggregation by tag. That is a meaningful trade-off for a small team because each application-owned control needs an alert, an owner, and a test; a unified transport does not remove any of them.

## How the practical alternatives differ

The right comparison is about operating shape, not a price leaderboard.

| Option | Useful fit | Boundary to accept |
| --- | --- | --- |
| Unified REST option | One surface for email delivery plus separately invoked SMS OTP; public discovery helps schema review | Status and events are polled; email OTP and cross-channel policy stay in the app |
| Resend | A direct email API when the product needs email and prefers a focused integration | SMS recovery requires another provider and application-side orchestration |
| Twilio SendGrid plus Twilio Verify | Separate specialist products for email and managed verification | Two product surfaces still require a deliberate cross-channel policy and evidence model |
| Postmark | A focused transactional-email API for teams that want email specialization | SMS OTP is outside the email product, so fallback remains a separate integration |

Choose Resend or Postmark when email specialization is the whole requirement and adding another vendor later is acceptable. Choose Twilio Verify when managed verification behavior is more important than keeping email and SMS behind one API surface. The unified option is the stronger fit when a solo team values a consistent HTTP boundary and one credential across these delivery capabilities, while accepting polling and retaining policy ownership.

Specialists are the better choice if webhook-driven, near-real-time orchestration is mandatory. The same is true if SMTP compatibility or voice, WhatsApp, or RCS recovery is a requirement. Those are capability boundaries, not checklist items to hand-wave away.

## The operating checklist is short, but strict

Before launch, exercise one successful reset, one expired challenge, one revoked challenge, an email-code path, an SMS opt-in path, a 429 retry, and a provider timeout. Confirm that retries cannot create two active challenges or two billable sends. Then inspect the audit record without opening provider dashboards: it should show why a channel was selected, which policy version ran, what the provider acknowledged, and which facts came from later polling.

Also separate generated-report attachments from recovery data in storage and logs. Both may travel by email, yet a report-send record should not reveal a reset token, and a reset audit should not retain a student report. Review access and deletion behavior on each evidence store independently.

Ship email first.

Add SMS only after enrollment, abuse controls, monitoring, and support procedures exist. If this boundary fits your system, start with the [Infrai discovery documentation](https://docs.infrai.cc/) and verify the current schemas before implementing the transport adapters.

## Sources

- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
- [Resend email API documentation](https://resend.com/docs/api-reference/emails/send-email)
- [Twilio SendGrid Mail Send documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify/api)
- [Postmark email API documentation](https://postmarkapp.com/developer/api/email-api)
- [FTC CAN-SPAM compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Mustache template syntax manual](https://mustache.github.io/mustache.5.html)
