# Attach PII-Safe Metadata to User Error Events by Release Environment and Request ID

TL;DR: For a flagged pricing-rule rollout, attach enough context to reconstruct the decision, not enough to reconstruct the person. Record a random request correlation ID, release identifier, deployment environment, stable error code, flag name and non-secret variant, processing region, and a coarse pseudonymous cohort when it is genuinely needed. Keep names, email addresses, raw account identifiers, payment data, authorization material, request bodies, and free-form user input out of the event. This produces a useful signal for rollback decisions while shrinking the privacy and security consequences of every copied error.

The governing trade-off is signal quality versus noise. A giant context object feels helpful during development, but it makes production failures harder to group and creates more data to govern. For a solo builder rolling out an LLM-assisted pricing rule, the useful question is narrow: did this release, flag variant, model operation, or region change the rate of a known failure class? The event schema should answer that quickly.

## What metadata should you attach to user error events for each release?

Imagine a fintech service introducing `pricing_rule_v3` behind a flag. The request enters an API, the application evaluates the flag, an AI step classifies supporting documents, and a deterministic pricing function produces the offer. On failure, the application emits one sanitized event to an error pipeline. Logs and traces may carry the same correlation identifiers, but the error event remains a deliberately small diagnostic record.

The event needs to support three decisions: pause the rollout, narrow the affected cohort, or keep shipping. That means dimensions with operational meaning belong in the schema. A release such as an immutable build ID distinguishes old and new code. An environment prevents staging failures from polluting production analysis. A flag variant separates control from treatment. A low-cardinality error code groups the same failure across stack traces. A processing-region field helps the operator apply the relevant retention and access policy. Do not use an email address as the join key. Do not put a full request URL into `request_id`; query strings often carry unexpected values. Generate an opaque identifier at the edge, propagate it, and validate its format before recording it. A request ID correlates system activity. It should not become a disguised customer record.

This is where restraint pays. If a field cannot change a rollout, debugging, or compliance decision, leave it out.

Identity can wait.

## Build the sanitizer before the error sender

The practical data flow is short: trusted application code creates a typed diagnostic context, a sanitizer allowlists fields and bounds their values, and only then does an adapter send the event to whichever sink is configured. The adapter never receives the original request, user object, model prompt, or payment payload. This boundary matters more than a long denylist because new input fields are excluded by default.

Here is a small TypeScript example. It uses a per-deployment HMAC key to derive an optional cohort token. That token can group repeated failures within a deliberately short analysis window without exposing the source account ID. Key rotation changes the token, which limits long-term linkability. The secret belongs in a secret manager and never in an event.

```ts
import { createHmac, randomUUID } from "node:crypto";

type DiagnosticInput = {
  release: string;
  environment: "production" | "staging";
  flag: { name: "pricing_rule_v3"; variant: "control" | "candidate" };
  region: "eu" | "us";
  errorCode: "MODEL_TIMEOUT" | "INVALID_MODEL_OUTPUT" | "PRICING_REJECTED";
  accountId?: string;
};

type ErrorEvent = {
  event_id: string;
  request_id: string;
  occurred_at: string;
  release: string;
  environment: DiagnosticInput["environment"];
  flag_name: DiagnosticInput["flag"]["name"];
  flag_variant: DiagnosticInput["flag"]["variant"];
  processing_region: DiagnosticInput["region"];
  error_code: DiagnosticInput["errorCode"];
  account_cohort?: string;
};

const SAFE_RELEASE = /^[a-zA-Z0-9._-]{1,64}$/;

function cohortToken(accountId: string, key: string): string {
  return createHmac("sha256", key).update(accountId).digest("hex").slice(0, 16);
}

export function makeErrorEvent(
  input: DiagnosticInput,
  cohortKey: string,
  requestId = randomUUID(),
): ErrorEvent {
  if (!SAFE_RELEASE.test(input.release)) throw new Error("Invalid release identifier");

  return {
    event_id: randomUUID(),
    request_id: requestId,
    occurred_at: new Date().toISOString(),
    release: input.release,
    environment: input.environment,
    flag_name: input.flag.name,
    flag_variant: input.flag.variant,
    processing_region: input.region,
    error_code: input.errorCode,
    ...(input.accountId
      ? { account_cohort: cohortToken(input.accountId, cohortKey) }
      : {}),
  };
}
```

The union types are intentional. They stop a stack message, prompt, or arbitrary upstream string from quietly becoming a grouping dimension. The release validator serves the same purpose. In a real service, validate the inbound request ID too or replace it at the trust boundary; never accept unlimited strings because an attacker can turn metadata into storage noise.

The optional cohort deserves suspicion. HMAC output is pseudonymous, not anonymous: the application can recompute it while it retains the source identifier and key. This example truncates the output to 16 hexadecimal characters, so it is a compact grouping token rather than a security boundary or proof of anonymity. Under GDPR, pseudonymized information that can be attributed to a person with additional information remains personal data. Treat the token accordingly, document its purpose, restrict access, and delete it when the debugging window closes. If rollout decisions work without it, omit it. I would ship the first rollout without this field and add it only when aggregate release and variant data cannot answer a documented operational question.

## Why not attach the whole user and request?

Because error systems replicate data. An event may move through an SDK buffer, collector, queue, index, alert, and support workflow. Each extra field expands the places where access, deletion, retention, and incident response must work. OWASP's logging guidance explicitly calls out data that should usually not be recorded directly, including session identifiers, access tokens, sensitive personal data, and payment-card data. GDPR Article 5 supplies a useful design test even outside Europe: personal data must be adequate, relevant, and limited to what is necessary for the purpose, and it must not be kept longer than necessary. A hash alone does not settle that question. Unsalted hashes of predictable identifiers can be guessed, while stable tokens can enable tracking across releases. Purpose and linkability matter.

US obligations vary by jurisdiction and sector, so an `us` value must not mean “privacy rules off.” Use it to select an approved storage location, retention schedule, and access policy defined by counsel and system owners. The event producer should not improvise law. It should enforce the smallest schema that operations has justified.

Some tempting fields are mostly noise. Raw exception messages may contain user input. Stack traces are useful for code faults, but they need source-map controls and a scrubber before export. Model prompts and outputs can contain financial documents or identifiers and should not ride along with an exception. Token counts, latency buckets, and model-operation names can be useful when defined as bounded metrics, but exact prompt content is a different dataset with a different purpose.

Prompts stay out.

## Separate correlation from identity

Use standard trace correlation where it exists. W3C Trace Context defines the `traceparent` and `tracestate` HTTP headers for propagating trace context, and OpenTelemetry's semantic conventions define common names across telemetry. A trace ID connects an error to spans; a request ID can connect application records when tracing is absent. Neither field should encode a user ID.

Keep identity lookup behind a separate, audited path. During an incident, an authorized operator may need to move from a pseudonymous cohort to an internal account record. That join should be exceptional, logged, time-bound, and unavailable to ordinary error dashboards. Most rollback decisions need aggregate counts by release, variant, region, and error code, not a person's identity. Cardinality is the other reason to separate them. Unique user IDs and raw URLs fragment groups and inflate indexes. Bounded dimensions make a comparison legible: candidate versus control, before versus after release, EU processing versus US processing. The result is cheaper to retain and faster to reason about without making price the argument.

## Operate the schema as part of the rollout

Before enabling the candidate variant, test the event builder with synthetic sensitive strings: an email address, an authorization header, a card-like number, and a sample prompt. Assert that none appears in the serialized event. Test allowed values as well. A privacy test that only checks redaction can pass while the sanitizer deletes the release field needed for rollback.

Then set the operational contract. Name an owner for each field, record its purpose, choose retention by event class and region, and restrict who can query cohort tokens. Sample repetitive failures after preserving accurate counters, because sampling raw events must not hide a rate change. Alert on a ratio or count tied to traffic volume rather than on every exception. Noise wins otherwise.

During rollout, compare the candidate and control variants using the same stable error taxonomy. If `INVALID_MODEL_OUTPUT` rises for the candidate in one release, the event has done its job: pause that variant and inspect the correlated trace without opening customer payloads. If the taxonomy changes mid-rollout, version it or keep a compatibility mapping so the apparent improvement is not merely renamed failures.

Finally, review the schema after the flag is retired. Remove flag dimensions that no longer drive decisions, rotate the cohort key on its documented schedule, verify expiry in every downstream store, and run a deletion drill. The best error context is temporary, boring, and sufficient.

## Sources

- https://eur-lex.europa.eu/eli/reg/2016/679/oj
- https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
- https://www.w3.org/TR/trace-context/
- https://opentelemetry.io/docs/specs/semconv/
- https://web.dev/articles/vitals
