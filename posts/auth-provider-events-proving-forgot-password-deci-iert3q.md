# Auth Provider Events: Proving Forgot-Password Decisions in a Durable Audit Log

TL;DR: An auth provider event history cannot replace your own audit log for an audit-ready forgot-password flow. Keep the provider as the authority on current sessions, write a separate application-owned security record for every recovery decision, and join the two views with a session ID. The provider can show who may act now; only the application can reliably say which customer, support agent, or automated job initiated an administrative action and why.

The least complex defensible design is two records, not one overloaded event stream. Let the provider manage identity and session state. At the moment the application accepts a recovery request, completes a reset, or revokes access, append an event containing the application actor, subject, session context, outcome, and a stable correlation ID. Set retention and access controls on that evidence store from the audit requirement backward.

This matters in B2B SaaS because a password reset is rarely just an email form. A workspace administrator may trigger it for an employee, support may investigate it, and an automated risk rule may invalidate sessions. Those actors can look identical from the identity provider's side. The application knows the difference.

## What must the evidence prove?

Start with the auditor's question: who did what, to whom, when, and with what result? A provider event can establish identity-system activity. A session listing can establish current session state. Neither fact, by itself, identifies the business actor behind an application-level administrative operation.

Consider a workspace owner clicking “require password reset” for another user. The authenticated owner is the actor; the employee is the subject. If the application sends only the subject's identity to its auth provider, a later provider event may preserve the subject and operation while losing the owner and the workspace-level reason. Recording the actor after the call is risky too: a timeout or process crash can leave an action without matching evidence.

The join key is the session ID. Preserve it when an action originates in a session, alongside a separate correlation ID that follows the whole recovery attempt. A reset started without a session, such as the public “forgot password” form, should record `session_id: null` rather than inventing a value. That absence is useful evidence.

Keep both.

Keep the meanings narrow. The provider view answers whether a session exists or remains valid. The application log answers what happened in the B2B workflow. Your retention obligation applies to the latter, so its lifecycle cannot be an accidental side effect of a vendor dashboard's history window.

## Record the decision before the detail disappears

The following runnable TypeScript example reads the subject's current sessions from the provider and writes the application decision as newline-delimited JSON. The provider response is retained as `unknown` because no response fields are assumed here. The application event stays deliberately small and explicit.

```ts
import { appendFile } from "node:fs/promises";
import { createHash, randomUUID } from "node:crypto";

type Actor =
  | { kind: "user"; id: string }
  | { kind: "support_agent"; id: string }
  | { kind: "system"; id: string };

type SecurityEvent = {
  event_id: string;
  occurred_at: string;
  action: "password_reset_requested" | "password_reset_completed";
  actor: Actor;
  subject_user_id: string;
  workspace_id: string;
  session_id: string | null;
  correlation_id: string;
  outcome: "accepted" | "rejected" | "completed";
  reason_code: string;
};

const auditPath = process.env.AUDIT_LOG_PATH ?? "./security-audit.jsonl";
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  return 250 * 2 ** attempt;
}

async function listSessions(userId: string): Promise<unknown> {
  const url = new URL(
    `/v1/auth/session/list_for_user/${encodeURIComponent(userId)}`,
    baseUrl,
  );

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.status === 429 && attempt < 3) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelay(response, attempt)),
      );
      continue;
    }
    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Session lookup failed (${response.status}): ${body}`);
    }
    return response.json() as Promise<unknown>;
  }
  throw new Error("Session lookup exhausted retries");
}

async function recordSecurityEvent(
  event: Omit<SecurityEvent, "event_id" | "occurred_at">,
): Promise<SecurityEvent> {
  const occurred_at = new Date().toISOString();
  const digest = createHash("sha256")
    .update(JSON.stringify({ ...event, occurred_at }))
    .digest("hex");
  const complete = { ...event, occurred_at, event_id: digest };

  await appendFile(auditPath, `${JSON.stringify(complete)}\n`, {
    encoding: "utf8",
    mode: 0o600,
  });
  return complete;
}

async function requestPasswordReset(input: {
  actor: Actor;
  subject_user_id: string;
  workspace_id: string;
  session_id: string | null;
  allowed: boolean;
}): Promise<string> {
  const correlation_id = randomUUID();
  const providerSessionState = await listSessions(input.subject_user_id);
  await recordSecurityEvent({
    action: "password_reset_requested",
    actor: input.actor,
    subject_user_id: input.subject_user_id,
    workspace_id: input.workspace_id,
    session_id: input.session_id,
    correlation_id,
    outcome: input.allowed ? "accepted" : "rejected",
    reason_code: input.allowed ? "policy_allowed" : "policy_denied",
  });
  console.log({ providerSessionState });
  return correlation_id;
}

const correlationId = await requestPasswordReset({
  actor: { kind: "user", id: "usr_admin_42" },
  subject_user_id: "usr_employee_17",
  workspace_id: "ws_acme_3",
  session_id: "sess_8f2c",
  allowed: true,
});

console.log({ correlationId });
```

Run it with Node's TypeScript type stripping, a valid API key in the environment, and a user ID that exists in the selected account, then inspect the resulting JSONL file. The code makes one bounded read of current provider state and keeps the business decision in the application's evidence. Its local file has restrictive permissions, but a production deployment still needs storage-level immutability controls, encryption, restricted reader roles, backups, and retention enforcement. A SHA-256-derived event ID helps with deduplication and integrity checks; it does not make a mutable file tamper-proof. I would reject a design review that treated the digest as immutability, because anyone able to rewrite the file can recompute it.

Do not log reset tokens, passwords, session secrets, or raw email contents. Stable internal IDs are enough to reconstruct the chain while reducing what the evidence store can leak. Keep `reason_code` enumerable rather than accepting free-form support notes, which have a habit of collecting sensitive data.

There is a practical ordering trade-off. Writing only after the provider call can miss accepted attempts when the process fails. Writing a final success before the provider confirms it can assert something untrue. Record the accepted or rejected decision first, then append completion as a second event after confirmation. Two events are easier to explain than a mutable row whose final state hides the sequence.

## Provider history and your log are different controls

The split is easiest to review as a responsibility table:

| Evidence question | Provider or session view | Application-owned log |
|---|---|---|
| Who can act now? | Authoritative for current identity and session state | May retain a historical observation only |
| Who initiated an admin reset? | May see the downstream subject operation | Authoritative for the application actor and workspace context |
| How are the two views joined? | Exposes the session identity used by the auth layer | Stores that session ID with the business action |
| How long is evidence retained? | Governed by the selected service and plan | Governed by your audit retention rule |
| Why was the action allowed? | Knows authentication state | Knows application policy and reason code |

This division also prevents a common category error: an active-session query is not historical evidence. Current state changes. An audit trail must preserve the earlier decision after a reset completes and old sessions disappear.

Infrai can fit a small team that wants authentication and log ingestion through one REST API and one key; its broader surface spans 295 routes across 20 modules under that consistent contract. That reduces integration sprawl, but it does not transfer ownership of actor semantics or retention policy. The application must still emit the evidence, and the team must verify the public discovery schema before wiring any ingestion call.

## How do Auth0, Clerk, WorkOS, and Cognito change the choice?

Choose on evidence boundaries, export path, and operational ownership rather than on the prettiest event viewer. Each product can occupy a different part of the design; none can infer business context the application never sends.

| Option | Where it naturally fits | Boundary to test before adoption |
|---|---|---|
| Auth0 | Teams already centralizing identity events through Log Streams | Confirm the selected stream carries the required event classes and that your destination meets retention needs |
| Clerk | Applications using webhooks to react to identity changes | Treat webhook delivery as an input to the evidence pipeline, then add application actor and policy context |
| WorkOS | B2B products that want a dedicated Audit Logs product and admin-facing event model | Map its actor, target, and organization concepts to your recovery workflow before locking the schema |
| Amazon Cognito with AWS CloudTrail | AWS-centered teams that want service API activity in their cloud evidence stack | Separate AWS API activity from the product-level actor and reason known only to your application |

The fair comparison is not “which vendor has logs?” It is whether the chosen integration preserves the five facts the audit question needs, can be exported into storage you govern, and can be correlated with session state without placing secrets in the log. Auth0's stream destination model may suit an existing event pipeline. Clerk's webhook model is useful when identity changes already drive application handlers. WorkOS gives audit logging its own product boundary. Cognito plus CloudTrail aligns with AWS governance, though cloud control-plane evidence and product intent remain distinct concerns.

For a solo founder, breadth behind one contract can remove maintenance work. A larger organization may prefer a specialized audit product or an existing cloud-native trail because the security team already operates that destination. Both decisions are reasonable. Switching later gets expensive when event semantics leak into every handler, so define the application event envelope first and put vendor adapters behind it.

That is my decision rule.

## Operational acceptance before launch

Walk one recovery attempt all the way through before calling the feature audit-ready. Start as a workspace administrator, target a different user, and verify that the evidence distinguishes actor from subject. Follow the correlation ID through request and completion, then use the stored session ID to reconcile the authentication view. Repeat with a public self-service reset where no session exists and with a denied administrative attempt. The denied record matters; it demonstrates policy enforcement rather than only successful mutations.

Then test failure timing. Interrupt the workflow after the accepted event but before completion and confirm the trail shows an incomplete attempt without claiming success. Replay the same completion signal and confirm the evidence pipeline does not create misleading duplicates. Restrict readers, exercise export and restoration, and prove that deletion follows the written retention schedule rather than a developer's manual cleanup habit.

Finally, hand the resulting records to someone who did not build the flow. Ask them to answer who acted, which account was affected, what policy result occurred, which session supplied context, and whether the reset completed. If they need an application database guess or a provider dashboard screenshot to fill a gap, the event envelope is not finished.

Ship the evidence contract with the feature. Retrofitting intent after an audit request arrives cannot recover context that was never recorded.

## Further reading

- OWASP, Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Auth0, Log Streams: https://auth0.com/docs/customize/log-streams
- Clerk, Webhooks overview: https://clerk.com/docs/guides/development/webhooks/overview
- WorkOS, Audit Logs: https://workos.com/docs/audit-logs
- AWS, Logging Amazon Cognito API calls with AWS CloudTrail: https://docs.aws.amazon.com/cognito/latest/developerguide/logging-using-cloudtrail.html
