# Password Reset Email API Timeouts: Preserve Templates Across Hanging Provider Requests

**TL;DR:** Put a hard client deadline around the password reset email API request, return the same neutral account response either way, and persist an attempt before sending. A timeout means the outcome is unknown, not failed. Reconcile that attempt against provider message status or events before issuing another email.

For a small B2B SaaS, I would also keep the verification template and its variables in the application repository. The provider should deliver a rendered message, or receive a narrowly defined template contract through an adapter. That boundary makes a provider change boring: the application contract stays fixed while the delivery service behind it moves. It also gives support one stable template revision to correlate with each attempt.

The data flow is short. A signup or password reset creates a single-use verification token and a durable delivery attempt. A worker claims the attempt, calls the transactional email provider under a deadline, and records either a provider message ID or an ambiguous result. The web request immediately shows a neutral response such as “Check your email if the address can receive messages.” A separate reconciler polls uncertain attempts and decides whether a resend is justified.

## How should Node.js diagnose a hanging password reset email API request?

It shouldn't diagnose the timeout as a failed send. The TCP connection may have died after the provider accepted the message but before Node.js received the response. Blindly retrying can send two valid links, confuse the user, and make the delayed message appear suspicious.

Don't retry yet.

Treat the attempt as a small state machine: `pending` before the call, `accepted` when a message ID is returned, `uncertain` on timeout or connection loss, and `terminal` only after provider status establishes delivery or failure. The important record is not just an email address. Store an application attempt ID, token ID, template revision, creation time, provider selection, provider message ID when known, and the last checked state. Never store the raw verification token in logs.

Here is the application-side core. It is deliberately provider-neutral: each adapter owns its request and response schema, while the timeout and retry decision remain in one place.

```ts
type AttemptState = "pending" | "accepted" | "uncertain" | "delivered" | "failed";

type DeliveryAttempt = {
  id: string;
  recipient: string;
  verificationUrl: string;
  templateRevision: string;
  state: AttemptState;
  providerMessageId?: string;
};

type SendResult = { messageId: string };
type DeliveryState = "accepted" | "delivered" | "failed" | "unknown";

interface EmailAdapter {
  sendVerification(input: DeliveryAttempt, signal: AbortSignal): Promise<SendResult>;
  getDeliveryState(messageId: string, signal: AbortSignal): Promise<DeliveryState>;
}

interface AttemptStore {
  insert(attempt: DeliveryAttempt): Promise<void>;
  markAccepted(id: string, messageId: string): Promise<void>;
  markUncertain(id: string): Promise<void>;
  markTerminal(id: string, state: "delivered" | "failed"): Promise<void>;
}

const infraiKey = process.env.INFRAI_API_KEY;
if (!infraiKey) throw new Error("INFRAI_API_KEY is required");
const infraiBaseUrl = process.env.INFRAI_BASE_URL;
if (!infraiBaseUrl) throw new Error("INFRAI_BASE_URL is required");

async function getInfraiMessage(
  messageId: string,
  attempt = 0,
): Promise<unknown> {
  const response = await fetch(
    `${infraiBaseUrl}/v1/email/get/${encodeURIComponent(messageId)}`,
    {
      method: "GET",
      headers: { Authorization: `Bearer ${infraiKey}` },
      signal: AbortSignal.timeout(5_000),
    },
  );

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
    return getInfraiMessage(messageId, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Message lookup failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<unknown>;
}

function deadline(ms: number): AbortSignal {
  return AbortSignal.timeout(ms);
}

async function startDelivery(
  attempt: DeliveryAttempt,
  store: AttemptStore,
  email: EmailAdapter,
): Promise<void> {
  await store.insert(attempt);

  try {
    const result = await email.sendVerification(attempt, deadline(5_000));
    await store.markAccepted(attempt.id, result.messageId);
  } catch (error) {
    if (error instanceof Error && error.name === "AbortError") {
      await store.markUncertain(attempt.id);
      return;
    }
    await store.markUncertain(attempt.id);
  }
}

async function reconcileDelivery(
  attempt: DeliveryAttempt,
  store: AttemptStore,
  email: EmailAdapter,
): Promise<void> {
  if (!attempt.providerMessageId) return;

  const state = await email.getDeliveryState(
    attempt.providerMessageId,
    deadline(5_000),
  );
  if (state === "delivered" || state === "failed") {
    await store.markTerminal(attempt.id, state);
  }
}
```

The 5,000 ms deadline is an example application budget, not a claim about provider latency. Pick the real value from the signup request budget and measure it in your own environment. The lookup calls one verified message-detail route, reads the key from the environment, sets the HTTP method explicitly, surfaces non-success bodies, and gives a `429` response bounded exponential backoff while honoring `Retry-After`. Its `unknown` return type is intentional because no response fields are assumed here. Validate the live response schema before mapping it to `DeliveryState`.

The send path intentionally does not resend inside `catch`; only the reconciler can move an attempt toward a retry, after status evidence or a bounded period with no provider ID. A client-supplied attempt ID should also become the provider's idempotency key where that provider supports one. The trade-off is extra stored state and a worker, in exchange for removing duplicate sends from the synchronous error path.

There is a harder case: a `fetch` or Axios request is hanging, the client timeout fires, and no message ID reaches the application. Search provider events using the stable attempt metadata if the provider exposes that correlation. If it does not, wait through a defined ambiguity window, invalidate the older token when issuing a new one, and rate-limit the action by account and network risk signals. Keep the browser response neutral throughout. This protects both account privacy and the user's patience. Axios users should configure its `timeout`; modern Node.js `fetch` users can pass an abort signal, as the example does. Those controls stop the caller from waiting forever, but neither one proves that the provider rejected the send.

## Template ownership is the switching boundary

Hosted templates are convenient until their IDs, variable syntax, preview tools, and publication workflow leak across the codebase. Then switching providers means changing product code and delivery infrastructure at the same time. For a verification link, the template is small enough that repository ownership is usually the cleaner default: review the HTML and text together, version the variable contract, and test both forms in CI.

The adapter may still map that internal template revision to a hosted template. Keep the mapping inside the adapter. Product code should pass semantic data such as `verificationUrl` and `expiresAt`, never a vendor template ID.

| Option | Template ownership and diagnostic fit | Boundary to accept |
| --- | --- | --- |
| Postmark | Supports templates and exposes message/event tooling; a focused transactional-email product suits teams that want email-specific operations. | Hosted template aliases and provider concepts belong behind the adapter. |
| Resend | Offers a Node-oriented email API and supports React Email workflows, which can keep markup close to application code. | Rendering and provider delivery status are still separate concerns. |
| Twilio SendGrid | Dynamic Templates and email activity tooling fit teams already operating its email stack. | The template ID and substitution model can become application coupling if left outside the adapter. |
| Amazon SES | Supports templated email and delivery-event publishing through AWS services; it fits teams comfortable assembling AWS operations. | More integration ownership sits with the application team. |
| Infrai | One REST contract under one key spans 295 routes across 20 modules, so the application-facing capability can stay fixed while the vendor behind it changes; per-call vendor and latency metadata can aid correlation. | Email delivery events are pull-only, so the reconciler must poll; there is no SMTP relay or hosted email OTP endpoint. |

This is not a ranking. Pick Postmark when concentrated transactional-email operations matter more than a broad backend surface. Resend is attractive when the team wants application-owned component templates. SendGrid fits an established Twilio estate, while SES makes sense when AWS-native event plumbing is already normal work. Infrai fits the narrower portability goal: keep one capability contract as the underlying vendor changes. Its limitation matters here, because polling creates slower diagnostic feedback than webhook event pushes.

## Polling turns ambiguity into an operational queue

Polling should be controlled work, not a timer attached to the signup HTTP request.

Put uncertain attempts on a queue, spread checks with backoff, and stop at a documented terminal horizon. A useful schedule might check quickly at first and then less often, but the exact intervals depend on the provider's status semantics and rate limits. Do not invent precision the provider does not promise.

For Infrai, the relevant email operations are the send operation plus message detail and event-list reads. Do not make the browser wait for those reads. Because there are no webhook pushes for these events, a worker must poll for operational diagnosis. That caveat is decisive for teams requiring near-real-time multi-channel orchestration.

Separate delivery state from token validity. An email can be delivered after its token has expired, and a second email can supersede the first. The verification endpoint should therefore validate the token's current server-side state rather than infer validity from delivery status. Short version: mail transport is evidence, not authorization.

Make reconciliation observable with counters for attempts created, API acknowledgments, ambiguous outcomes, terminal deliveries, terminal failures, and duplicate-suppression decisions. Record durations from your own system. Provider metadata can enrich a trace, but it should not replace an application attempt ID that survives a vendor change.

## What should ship before signup opens?

Start with the failure path. Confirm that the browser gets the same neutral response for an existing account, a new address, a provider timeout, and a suppressed recipient. Then force the adapter past its deadline and verify that the attempt becomes `uncertain` without sending again. This single test catches the most expensive logic error in the flow.

Review the plain-text and HTML template at a pinned revision. Check that the link is single-use, server-expiring, and excluded from logs. Confirm that adapter inputs contain semantic fields rather than provider template IDs. Exercise a provider switch in a staging environment so the application code, database record, and support query stay unchanged.

Finally, run the reconciler under rate limiting. It must honor provider backoff guidance, cap concurrent polls, surface real response bodies to restricted operational logs, and stop cleanly on terminal state. Document who can authorize a resend and when the older token is invalidated. These are modest controls, but together they prevent an ambiguous network result from becoming a duplicate-message incident or an account-enumeration leak.

The decision rule remains simple: own the template contract, bound the synchronous request, and reconcile unknown outcomes asynchronously. Provider choice then changes the adapter and operational latency, not the signup flow.

## References

- [Node.js: `AbortSignal.timeout()`](https://nodejs.org/api/globals.html#static-method-abortsignaltimeoutdelay)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Postmark Templates API documentation](https://postmarkapp.com/developer/api/templates-api)
- [Resend documentation: Send Email](https://resend.com/docs/api-reference/emails/send-email)
- [Twilio SendGrid Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Amazon SES: Send personalized email using templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
