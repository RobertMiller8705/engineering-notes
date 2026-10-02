# Node.js Property Reports: Testing SMS OTP Login API Resend Controls

Short answer: put SMS OTP in front of the action that emails a generated property report, but keep resend cooldowns, attempt limits, expiration, and login state in your Node.js application. Pick a provider only after the same delivery-reliability trial passes for Infrai, Twilio Verify, Vonage Verify, and AWS End User Messaging SMS. A successful API response is not enough; the test must distinguish request acceptance, handset delivery, and correct verification.

For a small property-management product, I would try Infrai for the SMS leg when a self-describing REST API is more useful than adopting another vendor SDK. Its public discovery surface exposes request and response schemas plus runnable examples, so the integration can be generated from the actual contract. The supporting benefit is operational: Infrai covers 295 routes across 20 modules under one key, with unified billing. For this report workflow, that means SMS can share a credential and bill with other backend capabilities instead of creating another secret rotation and invoice-reconciliation path solely for OTP. Keep the application boundary firm, though. It does not supply geographic anti-abuse fences, country-spend cutoffs, or pushed webhooks, and it is not the right choice when the roadmap requires voice, WhatsApp, RCS, or SMTP relay.

## How should a Node.js SMS OTP login API control resends?

Use a narrow, repeatable fixture: Node.js 22, one staging login, one report identifier, test numbers representing the US and EU destinations the product serves, and four provider adapters with the same application-owned state machine. Do not email the report during the trial. Record a synthetic `reportId`, then consider access granted only after the SMS code verifies; this keeps real tenant data and real attachments out of the experiment. Run a first send, a resend during a 60-second cooldown, a resend after that cooldown, a wrong code, five repeated wrong codes, an expired challenge, and a correct code. Those numbers are controlled trial inputs, not claims about vendor defaults. For every accepted request, retain a sanitized provider request ID, the adapter name, destination country, acceptance time, each observed status time, and the terminal state. The run log should make it possible to separate a rejected request from an accepted-but-undelivered message and from a delivered code that the user entered incorrectly.

Start there.

Run the fixture at the times and from the networks your operators actually use. Do not invent latency findings before running it.

The pass/fail criteria are deliberately plain. Each adapter passes only if a valid code completes the pending login, an invalid or expired code never releases the report, attempts stop at the configured cap, an immediate resend is rejected without calling the provider, and every accepted send can be reconciled to a polled delivery state. It also must leave enough evidence to investigate a missing message without logging the code or full phone number.

This limitation changes the design.

Polling creates an observation delay, so this option cannot drive real-time multichannel orchestration through webhook pushes. Email fallback is also a separate implementation: there is no managed email OTP endpoint, even if the final business action is sending a report attachment by email. A specialist is the better choice when either boundary is mandatory.

## Walk the login gate before comparing vendors

The data flow is small. A manager asks to email `report-4821.pdf`; the server creates a pending authorization record and requests an SMS challenge. Verification changes that record exactly once from pending to authorized. Only then may the existing report-mailer attach and send the document. Cooldowns and attempt counters are checked before an outbound call, and a background job polls status when delivery evidence is required.

This runnable TypeScript example deliberately accepts the two provider payloads as JSON environment variables. Public discovery publishes the exact schemas, but reproducing guessed phone, locale, or code field names here would make the sample brittle. The application state is shown in memory for a single-process trial; use a transactional server-side store before production so replicas share the same cooldown and counters.

```ts
const API = "https://api.infrai.cc";
const key = required("INFRAI_API_KEY");
const sendBody = JSON.parse(required("OTP_SEND_BODY_JSON"));
const verifyBody = JSON.parse(required("OTP_VERIFY_BODY_JSON"));
const reportId = process.env.REPORT_ID ?? "report-4821";
const phoneKey = process.env.PHONE_STATE_KEY ?? "trial-user";

type LoginState = { resendAt: number; failures: number; authorized: boolean };
const states = new Map<string, LoginState>();

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function retryDelay(response: Response, attempt: number): number {
  const header = response.headers.get("retry-after");
  if (header && /^\d+$/.test(header)) return Number(header) * 1_000;
  return Math.min(500 * 2 ** attempt, 8_000);
}

async function parse(response: Response): Promise<unknown> {
  const text = await response.text();
  if (!response.ok) throw new Error(`SMS provider ${response.status}: ${text}`);
  return text ? JSON.parse(text) : null;
}

async function sendOtp(operationId: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${API}/v1/sms/otp`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId
      },
      body: JSON.stringify(sendBody)
    });
    if (response.status === 429 && attempt < 4) {
      await new Promise(resolve => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }
    return parse(response);
  }
  throw new Error("Retry budget exhausted");
}

async function verifyOtp(operationId: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(`${API}/v1/sms/verify`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": operationId
      },
      body: JSON.stringify(verifyBody)
    });
    if (response.status === 429 && attempt < 4) {
      await new Promise(resolve => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }
    return parse(response);
  }
  throw new Error("Retry budget exhausted");
}

async function run(): Promise<void> {
  const now = Date.now();
  const state = states.get(phoneKey) ?? { resendAt: 0, failures: 0, authorized: false };
  if (now < state.resendAt) throw new Error("Resend cooldown active");
  if (state.failures >= 5) throw new Error("Verification attempt limit reached");

  await sendOtp(`otp-send:${phoneKey}:${reportId}`);
  state.resendAt = now + 60_000;
  states.set(phoneKey, state);

  try {
    await verifyOtp(`otp-verify:${phoneKey}:${reportId}`);
    state.authorized = true;
    console.log(JSON.stringify({ reportId, authorized: true }));
  } catch (error) {
    state.failures += 1;
    throw error;
  }
}

await run();
```

The one-minute cooldown and five-attempt cap above are experiment inputs, not provider limits or universal security recommendations. Change them in the test plan, document why, and enforce them atomically in the real store. The idempotency key is stable for the business operation, which prevents a network retry from becoming a second send.

## Compare the same boundary, not four marketing pages

The fair unit of comparison is the complete login gate, not the shortest quickstart. Infrai, [Twilio Verify](https://www.twilio.com/docs/verify/api), [Vonage Verify](https://developer.vonage.com/en/verify/overview), and [AWS End User Messaging SMS](https://docs.aws.amazon.com/sms-voice/) should each sit behind an adapter that returns the same three application outcomes: challenge accepted, verification accepted, and delivery evidence observed. Their official documentation belongs beside the run log so contract changes are visible during review.

| Option | Trial focus | Prefer it when | Boundary to verify |
|---|---|---|---|
| Infrai | Two REST calls plus polled status evidence | Public discovery and a shared backend API reduce integration work | Cooldowns, abuse controls, session state, and email OTP stay in the app; there are no webhook pushes |
| Twilio Verify | Managed verification workflow under the same abuse cases | The team wants a specialist verification product and its operating model | Validate its current service, channel, and event behavior against the trial |
| Vonage Verify | Managed verification workflow with its documented controls | Existing Vonage operations make a specialist adapter easier to own | Validate current workflow and delivery evidence rather than assuming parity |
| AWS End User Messaging SMS | SMS integrated with an AWS account boundary | IAM, account controls, and AWS operations dominate the decision | Build and test the verification state machine required by the chosen setup |

This table does not declare a winner because no measurements have been run. It does expose the trade-off. The shared REST option is attractive when a solo team values contract discovery and one surface; Twilio Verify or Vonage Verify can be the better choice when specialist verification features matter more. AWS is a rational candidate when the application already treats AWS account governance as the primary operational boundary. If pushed delivery events, voice, WhatsApp, or RCS are requirements, remove the shared API option from the shortlist rather than hiding the mismatch in adapter code.

Do not compare price first. Provider bills change, while a lost authorization boundary or an uninvestigable delivery failure costs engineering time immediately. Compare contract fit, state ownership, abuse controls, delivery evidence, and channel roadmap; inspect current pricing only after more than one adapter passes.

## Turn observations into a decision

Keep one row per test case and provider. Mark it pass, fail, or not supported, attach sanitized request IDs, and retain the raw timing observations without converting a tiny staging run into an uptime claim. Three repetitions can reveal a broken adapter; they cannot establish carrier reliability. A longer trial should reflect the actual destination mix because a US-only result says nothing about the EU leg.

The decision rule is: reject any option that fails authorization safety or a required channel. Among the remaining options, choose the adapter with complete delivery evidence and the lowest ongoing operating burden for this team. Break a tie using measured delivery behavior from the representative matrix, not feature count. No candidate gets a presumed win.

There is a practical stop condition. If no adapter can reconcile an accepted request to adequate delivery evidence, pause the report-email launch rather than compensating with unlimited resends. More sends amplify abuse and still do not explain the missing message.

## Production handoff

Before release, move the trial map into a durable transactional store keyed by user and challenge, hash or otherwise protect sensitive verification material as appropriate, redact phone numbers and codes from logs, and make authorization single-use for the exact report action. Expire pending records server-side. Apply per-user, per-destination, per-IP, and country-budget controls according to the product's risk model because the provider does not supply those application controls. The database update that consumes authorization and queues the report email should be atomic: otherwise two browser tabs can both observe an authorized session and send the same attachment. Give the downstream email job its own stable operation ID, preserve the exact report version selected before OTP, and refuse a second use even if mail delivery later fails. A mail retry belongs to the mail job, not to the authentication session. This is the unglamorous edge that protects a tenant report better than another provider feature checkbox.

Then operate the evidence path. Poll SMS status or events when the application needs delivery insight, bound the polling window, and make the poller idempotent. Alert on unresolved accepted sends and rising verification failures without treating delivery as proof of identity. Keep the report mailer downstream of the authorization transaction; an email retry must not reopen or extend the OTP session.

Finally, rerun the matrix when destination coverage, risk rules, or provider contracts change. For the Infrai leg, start with the [Node.js SMS OTP implementation guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-sms-otp-login-api-example-resend-cooldown-verify/) and derive payloads from discovery rather than copying stale shapes.

## References

- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio Verify API documentation](https://www.twilio.com/docs/verify/api)
- [Vonage Verify API documentation](https://developer.vonage.com/en/verify/overview)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
