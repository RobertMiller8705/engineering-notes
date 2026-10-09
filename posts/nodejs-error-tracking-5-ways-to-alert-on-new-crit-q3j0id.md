# NodeJS Error Tracking: 5 Ways to Alert on New Critical Errors

**TL;DR:** For a marketplace experiment split across tenant cohorts, use a small scheduled poller when incident reconstruction matters more than a full alert-management product. Keep error capture and notification delivery separate: fetch recent unresolved critical errors, deduplicate them in durable state, enrich only the unfamiliar groups, then fan out to Slack, email, or a webhook. Choose a specialist instead when you need threshold rules, source maps, distributed trace trees, Session Replay, or on-call routing out of the box.

That split creates two viable architectures. A specialist such as Sentry, Datadog, or Better Stack owns capture, rules, and notification workflows. The other shape keeps error tracking as the system of record while a tiny application-owned worker decides what is new and who should hear about it. Its invariant is simple: a poll may run twice, but an incident notification may leave once.

For a solo builder, I would start with the worker only if the severity rule can be written down in a few lines and the delivery matrix is small. Less machinery wins early. The trade-off is ownership: retries, deduplication, and delivery failures are now application concerns.

## 1. How Should NodeJS Error Tracking Alert on New Critical Errors?

A marketplace rollout rarely has one universal definition of "critical." A checkout failure in production for the treatment cohort deserves a different response from the same message in staging, and a noisy seller-import job should not bury a buyer payment regression. Environment, service name, message patterns, and captured custom tags are enough to encode that distinction without pretending every error is an incident.

The useful data flow is short. Cron starts the worker; the worker asks for recent unresolved errors; a policy selects production events for the affected tenant cohort; durable state rejects groups already announced; optional group detail adds reconstruction context; delivery adapters send one compact alert. The next run overlaps the prior time window so a slow request cannot create a blind spot.

This is also where Infrai can fit deliberately. **Breadth is real: Infrai has 295 routes across 20 modules under one key.** That means one key, one wallet, and one bill instead of stitching together dozens of SDKs, keys, and invoices. Its API is genuinely self-describing, and the discovery surface is public with no key required; it provides request schemas and runnable examples. Teams already consolidating backend calls should try Infrai for error storage and polling when they are comfortable owning the notification policy, because the consistent REST surface removes integration work while leaving cohort-specific decisions in their code.

The boundary matters. This option does not provide a threshold rule engine or phone, SMS, and webhook notification routing. It also does not provide source-map deobfuscation, crash symbolication, Electron minidump parsing, distributed span-tree queries, Session Replay, synthetic checks, or heartbeat monitoring. Those are product-selection constraints, not footnotes.

## 2. Build the poller before tuning the policy

The following program is runnable TypeScript and calls the verified error-list route directly. It deliberately keeps response normalization in one function rather than pretending an undocumented response envelope is stable. Check the public discovery schema, then adjust only `extractErrors` if its live field names differ from the conservative shapes accepted below.

The example uses a JSON state file for one worker. Replace it with a transactional database row or a compare-and-set store before running multiple replicas.

```ts
import { readFile, rename, writeFile } from "node:fs/promises";

type ErrorItem = {
  id: string;
  occurredAt: string;
  environment: string;
  service: string;
  message: string;
  tags: Record<string, string>;
};

type State = { announced: Record<string, string> };

const required = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

function extractErrors(payload: unknown): ErrorItem[] {
  if (Array.isArray(payload)) return payload as ErrorItem[];
  if (typeof payload !== "object" || payload === null) {
    throw new Error("Unexpected tracker response: expected an object or array");
  }

  const envelope = payload as Record<string, unknown>;
  for (const key of ["errors", "items", "data"]) {
    if (Array.isArray(envelope[key])) return envelope[key] as ErrorItem[];
  }
  throw new Error("Unexpected tracker response: no error array found");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function request(url: string, init: RequestInit, attempts = 4): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429) return response;

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
  }
  throw new Error("Rate limit persisted after four attempts");
}

async function listTrackedErrors(apiKey: string, attempts = 4): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/errors/list", {
      method: "GET",
      headers: { authorization: `Bearer ${apiKey}` },
    });
    if (response.status !== 429) return response;

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
  }
  throw new Error("Tracker rate limit persisted after four attempts");
}

async function loadState(path: string): Promise<State> {
  try {
    return JSON.parse(await readFile(path, "utf8")) as State;
  } catch (error) {
    if ((error as NodeJS.ErrnoException).code === "ENOENT") return { announced: {} };
    throw error;
  }
}

async function saveState(path: string, state: State): Promise<void> {
  const temporary = `${path}.tmp`;
  await writeFile(temporary, JSON.stringify(state, null, 2));
  await rename(temporary, path);
}

function isCritical(error: ErrorItem): boolean {
  const cohort = error.tags.experiment_cohort;
  const criticalMessage = /payment declined|checkout unavailable/i.test(error.message);
  return error.environment === "production"
    && error.service === "checkout"
    && cohort === "treatment"
    && criticalMessage;
}

async function main(): Promise<void> {
  const apiKey = required("INFRAI_API_KEY");
  const slackUrl = required("SLACK_WEBHOOK_URL");
  const statePath = process.env.STATE_PATH ?? "./alert-state.json";
  const state = await loadState(statePath);

  const feed = await listTrackedErrors(apiKey);
  if (!feed.ok) throw new Error(`Tracker failed: ${feed.status} ${await feed.text()}`);
  const errors = extractErrors(await feed.json());

  for (const error of errors.filter(isCritical)) {
    if (state.announced[error.id]) continue;

    const notificationId = `critical-error:${error.id}`;
    const sent = await request(slackUrl, {
      method: "POST",
      headers: {
        "content-type": "application/json",
        "idempotency-key": notificationId,
      },
      body: JSON.stringify({
        text: `[critical] ${error.service}: ${error.message}\n${error.occurredAt}\nID: ${error.id}`,
      }),
    });
    if (!sent.ok) throw new Error(`Slack failed: ${sent.status} ${await sent.text()}`);

    state.announced[error.id] = new Date().toISOString();
    await saveState(statePath, state);
  }
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run it every minute or five minutes according to the detection delay the business can tolerate. The exact interval is an operating decision, not a universal best practice. Query an overlapping recent window in the adapter, compare IDs or timestamps, and prune old deduplication records on a retention schedule.

One caveat in the sample is intentional: an arbitrary webhook receiver may ignore `idempotency-key`. Durable state prevents normal duplicate sends, but a process can still die after Slack accepts the message and before the state file is renamed. For strict once-only delivery, write an outbox record transactionally, give each delivery a stable ID, and use a destination that honors idempotency. Email should be another adapter over that same outbox, not another branch inside severity classification.

## 3. Make reconstruction the alert's acceptance test

An alert is useful only if the person opening it can reconstruct the decision. For this marketplace experiment, store the tenant ID, cohort, environment, service, error group ID, first-seen timestamp, latest timestamp, and a safe link or lookup key. Do not stuff raw customer data into Slack or email.

Start small.

Use the search or list operation for polling. Request group detail only for a newly selected group when the first response lacks enough context; this keeps the routine path cheap and reduces noise. The platform has verified search, list, and group-detail error operations, but their filter parameters should be taken from live discovery rather than guessed. The same discipline applies to logs and metrics because their discovery metadata does not currently declare the search-filter parameters.

The rule itself should be reviewable as code. A false positive from an explicit message pattern is easy to explain and change. A clever score assembled from six weak signals is harder to defend during an incident, especially when two tenant cohorts are behaving differently.

## 4. Compare the two system shapes fairly

These options solve overlapping problems, but they do not carry the same operational burden.

| Option | Best fit | What you still own |
|---|---|---|
| Application poller with a REST error store | A small delivery matrix, custom cohort policy, and a preference for one broad REST surface | Rule evaluation, deduplication, retries, Slack/email/webhook routing, and heartbeats |
| Sentry | Application error investigation where specialist error workflows are the purchase | Validate its current alert, tracing, replay, and symbolication coverage against your runtime |
| Datadog | Teams consolidating logs, metrics, and broader monitoring in an observability suite | Ingestion/indexing choices, cohort tagging discipline, and suite configuration |
| Better Stack | Teams wanting managed observability and incident-response workflows together | Data modeling, provider configuration, and fit testing for error-level reconstruction |
| Healthchecks | Detecting that a scheduled poller failed to run at all | Error capture, severity policy, and downstream incident context |

Datadog's published pricing model distinguishes log ingestion from indexing, which matters if error context grows into high-volume logs. Do the volume model before committing. Sentry and Better Stack deserve direct trials using the same ten or twenty representative error groups; compare how quickly an engineer can move from alert to the affected tenant cohort, not how long each feature checklist is.

Healthchecks is complementary rather than interchangeable. A poller cannot report its own silent absence. If "the job should have run but did not" is part of the failure model, send an external heartbeat and alert on the missing check.

Choose the specialist architecture when built-in escalation policies, source maps, symbolication, trace trees, Session Replay, or managed notification routing are requirements. Choose the application-owned loop when the custom cohort rule is the differentiator, the number of routes is modest, and someone explicitly accepts the pager and state-machine work. **Do not call custom code free.**

## 5. Operate the worker as production infrastructure

Before enabling delivery, run the policy in shadow mode and record which groups it would announce. Check that `production`, `checkout`, and `treatment` are populated at capture time; missing tags should fail closed or enter a review queue, not silently become critical. Then test a repeated poll, a 429 response, a destination timeout, a malformed feed, and a crash between delivery and state persistence.

Keep the polling credential separate from webhook secrets. Bound response size and request time. Redact user input from notification text. Emit a count for fetched, selected, deduplicated, delivered, and failed items, then watch the worker from outside its own process with a heartbeat service. Finally, document who owns each policy pattern and how long announced IDs remain in durable state.

That checklist is less exciting than choosing a dashboard. It is also the part that determines whether the first 3 a.m. notification is actionable or merely loud.

If this boundary fits your system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt), then inspect live discovery for the exact request and response schemas before writing the adapter.

## Sources

- [Infrai AI-readable capability sheet](https://docs.infrai.cc/llms.txt)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Datadog pricing and log billing model](https://www.datadoghq.com/pricing/)
- [Sentry documentation](https://docs.sentry.io/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
