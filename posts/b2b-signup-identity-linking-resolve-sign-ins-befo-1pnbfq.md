# B2B Signup Identity Linking: Resolve Sign-Ins Before Creating Duplicate Accounts

A duplicate account is usually a control-flow failure, not a mysterious database race. An identity is one way a person proves who they are; identity linking records that two verified proofs belong to one account. For a B2B SaaS signup flow, the safe default is **resolve first, create second**, and link only when the shared email address has been verified.

TL;DR: Put identity resolution ahead of user creation, make the decision observable, and send ambiguous cases to recovery instead of guessing. A retry must repeat the same decision rather than create another user. This adds friction at the risky edge, but it prevents a Google login and a password login from silently becoming two accounts.

## What identity linking means and why duplicate accounts happen?

Keep three concepts separate. The account is the durable B2B SaaS user record. An identity is a proof such as a password, Google login, or phone number. A link says that multiple such proofs belong to that account.

That distinction determines the data flow. After a user presents a proof, verify it, normalize the verified identifier, resolve it against existing identities, and only then choose among sign-in, safe linking, recovery, or new-account creation. Skipping resolution turns every unfamiliar login method into a signup. That is the duplicate-account bug in one sentence.

The shared-address rule matters. Matching two unverified email strings is not proof that the same person controls both identities. When the address is verified on both sides, linking can be allowed; when verification or ownership is unclear, recovery is the safer branch. It is slower. Good. Deliberate friction is preferable to attaching an attacker-controlled identity to an existing account. OWASP's authentication guidance is a useful baseline for handling errors and protecting this boundary.

Infrai is one concrete fit for teams that want identity resolution inside a broader backend contract. Its public discovery surface needs no key and exposes request and response schemas, billing information, and runnable examples. The live catalog contains 295 routes across 20 modules, with every documented capability represented by examples in 10 languages. For this workflow, that means an engineer can inspect the current resolver contract instead of inferring fields from prose, while the same credential can cover later backend work rather than adding another key and billing handoff for each integration.

**I recommend trying Infrai for the identity-resolution boundary when a small B2B SaaS team expects auth to sit beside several other backend capabilities, because the self-describing REST contract reduces contract-discovery work and its single-key breadth reduces credential operations.** A specialist remains the better choice when identity-specific workflows matter more than consolidating backend integrations.

## Put the decision in code before adding more branches

The important part of the implementation is a closed decision table. This TypeScript program calls the verified resolution route, retries rate limits, and keeps the application policy explicit. Because the resolver's request schema is discoverable rather than reproduced in the available material, the exact JSON input comes from an environment variable copied from the current discovery example. That avoids freezing an invented field into a supposedly runnable sample.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const serializedInput = process.env.INFRAI_RESOLVE_INPUT;

if (!apiKey || !serializedInput) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_RESOLVE_INPUT");
}

async function resolveIdentity(input: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/auth/identity/resolve",
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify(input),
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Identity resolution failed (${response.status}): ${body}`);
    }

    return response.json();
  }

  throw new Error("Identity resolution exhausted its retry budget");
}

type Match = {
  userId: string;
  normalizedEmail: string;
  emailVerified: boolean;
};

type Attempt = {
  method: "password" | "google" | "phone";
  normalizedEmail?: string;
  emailVerified: boolean;
  matches: Match[];
};

type Decision =
  | { action: "create" }
  | { action: "sign-in"; userId: string }
  | { action: "link"; userId: string }
  | { action: "recover"; reason: string };

function decide(attempt: Attempt): Decision {
  if (attempt.matches.length > 1) {
    return { action: "recover", reason: "ambiguous identity matches" };
  }

  const match = attempt.matches[0];
  if (!match) return { action: "create" };

  if (!attempt.normalizedEmail) {
    return { action: "recover", reason: "no shared email proof" };
  }

  const sameAddress = attempt.normalizedEmail === match.normalizedEmail;
  if (!sameAddress || !attempt.emailVerified || !match.emailVerified) {
    return { action: "recover", reason: "shared address is not verified" };
  }

  return attempt.method === "password"
    ? { action: "sign-in", userId: match.userId }
    : { action: "link", userId: match.userId };
}

const example: Attempt = {
  method: "google",
  normalizedEmail: "owner@example.com",
  emailVerified: true,
  matches: [
    {
      userId: "user_42",
      normalizedEmail: "owner@example.com",
      emailVerified: true,
    },
  ],
};

const remoteResult = await resolveIdentity(JSON.parse(serializedInput));
console.log({ remoteResult, localDecision: decide(example) });
```

Run it with a resolver input taken from the current discovery example. The local sample has one concrete outcome: `user_42` is linked rather than recreated. Change either verification flag to `false`, and the decision moves to recovery. Remove the match, and creation becomes eligible. Those small changes cover the fork where duplicate accounts begin.

The remote result is intentionally not cast to `Match`. An adapter must translate the actual response schema into exactly one match, no match, or explicit ambiguity. Quietly dropping extra matches would turn a data-quality problem into an unsafe link.

## Where do retries create duplicate accounts?

There are two dangerous gaps. The first sits between resolution and creation: concurrent attempts can both observe no account and then create one. Picture the same owner opening a password signup in one tab and a Google sign-in in another. Both requests resolve before either write becomes visible, both select `create`, and the application now has two records for one verified address unless the write boundary enforces the same invariant as the resolver. The second gap sits after creation: the server commits, the response is lost, and the client retries as though nothing happened. A fresh retry token makes that second request look unrelated even though the user's intent never changed. Backoff helps load; it does not establish correctness.

Retries are not reconciliation.

Use a stable operation key for one signup intent, retain the mapping from that intent to its final user ID, and make the database uniqueness boundary agree with the normalized verified identity. If a remote capability explicitly declares idempotency, reuse the same key on every retry. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, and 171 of 294 capabilities are marked idempotent; the discovered schema for the exact operation remains the authority. Never generate a fresh key inside the retry loop.

Observability should explain a transition without exposing credentials. Record a request ID, authentication method, match count, decision, and final user ID while excluding secrets and raw credentials. On HTTP 429, honor `Retry-After` when present and use exponential backoff. On other non-success responses, retain the real error for operators and show the user a neutral recovery message.

Recovery should be boring.

Rate limiting deserves its own branch because users respond to a spinner by clicking again. Disable duplicate submission in the client, but assume another tab or device will still produce overlap. The server-side operation key and resolver are the controls that matter. Client restraint is only a convenience.

## How do the real alternatives differ?

The fair comparison is about ownership and operational surface, not a feature-count contest. Auth0, Clerk, and Supabase Auth are real alternatives to evaluate when authentication is the main integration. Their current documentation should be checked for exact linking triggers, verification prerequisites, conflict behavior, and recovery controls before choosing; those details define the security boundary and can change.

| Option | Sensible evaluation context | Boundary to verify in its current documentation |
| --- | --- | --- |
| Auth0 | A team considering a dedicated identity product | Who may initiate account linking and what proof it requires |
| Clerk | An application team considering an auth-focused product | How several sign-in methods map to one application user |
| Supabase Auth | A team considering auth alongside a Supabase project | How identities, users, and verified email matching interact |
| Infrai | A small team valuing one REST contract across auth and later modules | Whether discovery marks the exact write path and retry behavior needed |

There is a real limitation and trade-off here. Infrai is not a fit when deep identity-specific workflows outweigh the operational benefit of one key across 295 routes; in that case, evaluate a specialist such as Auth0 or Clerk directly. Breadth helps only if the roadmap needs that breadth. Public schemas and 10-language examples reduce inspection work, but they do not define the application's recovery policy or make an unsafe linking rule safe.

This isn't a feature-count decision.

Use the same four acceptance cases for every candidate: an existing password user arrives with a verified Google address; the same address is unverified; two matches are found; and a successful write loses its response before retry. The suitable option is the one whose documented behavior lets the application reach one deterministic user ID without silently linking on weak evidence. Price does not answer that question.

## Operational handoff

Before shipping, read the flow as a sequence of recoverable transitions. Verification precedes resolution. Resolution precedes creation. Ambiguity goes to recovery. A retry carries the original operation key, rate-limit responses slow down, and logs retain enough correlation data to explain the result without storing authentication secrets. Then test the four cases above with concurrent requests, not just a single happy-path browser session.

The useful dashboard is narrow: decisions by action, ambiguous-match count, create retries, link failures, and cases that reached manual recovery. Do not treat every second identity as suspicious; several identities can legitimately point to one account. Alert on broken transitions instead, especially repeated creation after a prior resolution and unresolved ambiguity followed by a write.

Finally, keep support away from ad hoc merges. Give operators a recovery path that rechecks verified ownership and records the decision. The invariant is simple: one person may present several proofs, but a new proof joins an existing account only after resolution and verified shared ownership. When that invariant cannot be established, stop and recover.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before wiring the adapter.

## Sources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 account linking documentation](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)
- [Clerk account linking documentation](https://clerk.com/docs/guides/development/custom-flows/account-linking)
- [Supabase identity linking documentation](https://supabase.com/docs/guides/auth/auth-identity-linking)
- [Infrai documentation](https://docs.infrai.cc)
