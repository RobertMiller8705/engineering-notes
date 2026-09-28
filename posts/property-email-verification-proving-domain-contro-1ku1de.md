# Property Email Verification Proving Domain Control (TXT or CNAME Under Shared DNS)

TL;DR: Use a dedicated TXT record to prove that a property-management customer controls a domain, then evaluate mail authentication separately. Use a CNAME proof only when the customer can delegate an otherwise unused hostname and the application benefits from controlling what that hostname resolves to later. Never ask for both record types at the same owner name: a CNAME is exclusive there, while a TXT proof can share a name with other non-CNAME data. For an independent builder, TXT is usually the least complex ownership proof. The harder and more valuable work is collecting deliverability evidence after verification.

A customer connecting `harborview.example` to a property platform may want branded leasing notices, maintenance updates, and rent reminders. The onboarding flow has two distinct jobs: establish authority to configure the domain, then determine whether the domain is ready to send authenticated mail. One green check cannot honestly represent both.

The data flow is small. The application creates a random, single-use token bound to a tenant and hostname. It tells the customer which DNS record to publish, queries DNS through the application's resolver path, compares the observed value with the expected value, and stores the evidence and observation time. A separate evaluator reads the domain's mail-authentication records and reports their state without silently treating ownership as deliverability.

Keep those states apart.

## Should TXT or CNAME verification prove domain control?

The owner name is the detail that causes the conflict. Suppose onboarding asks a customer to create both `verify.harborview.example TXT ...` and `verify.harborview.example CNAME ...`. Those instructions are internally inconsistent. A CNAME makes that owner name an alias and cannot coexist with other record data at that same name. The token may be correct, yet the requested DNS state cannot be represented.

Names matter.

This gets muddier in a property business because domain administration is often shared among a registrar account, an IT contractor, and a marketing agency. A person may have enough access to add a TXT record but may not be authorized to redirect a hostname. That is still useful evidence of control. It is not evidence that the same person understands, or should alter, the mail path.

**Treat exclusivity as a schema rule, not a retryable error.** Before showing instructions, reserve one owner name for one proof method. A TXT-oriented name such as `_property-proof.harborview.example` keeps the verification artifact away from web and mail hostnames. If the application requires CNAME behavior, allocate a different, dedicated label and first check that no data already exists at that owner name.

There is another boundary worth making explicit. DMARC is published as a TXT record at a `_dmarc` name and expresses a domain owner's requested mail-handling policy along with reporting parameters. It is deliverability evidence, not an application ownership token. Reusing `_dmarc.harborview.example` for product verification would mix unrelated lifecycles and permissions. Don't do it.

## Put the conflict rule in code before writing setup copy

The example below models the decision without tying it to a DNS provider. It is deliberately strict: an occupied name cannot become a CNAME target, and an existing CNAME prevents adding TXT data at the same name. It also keeps ownership and mail evidence as separate results.

```ts
type RecordType = "A" | "AAAA" | "CNAME" | "MX" | "TXT";

type DnsRecord = {
  name: string;
  type: RecordType;
  values: string[];
};

type ProofPlan =
  | { ok: true; method: "TXT" | "CNAME"; name: string; value: string }
  | { ok: false; reason: string };

function planDomainProof(
  zone: string,
  token: string,
  records: DnsRecord[],
  method: "TXT" | "CNAME",
): ProofPlan {
  const label = method === "TXT" ? "_property-proof" : "property-proof";
  const name = `${label}.${zone}`.toLowerCase();
  const atName = records.filter((record) => record.name.toLowerCase() === name);

  if (method === "CNAME" && atName.length > 0) {
    return { ok: false, reason: `CNAME owner name is already in use: ${name}` };
  }

  if (method === "TXT" && atName.some((record) => record.type === "CNAME")) {
    return { ok: false, reason: `TXT cannot share a CNAME owner name: ${name}` };
  }

  return method === "TXT"
    ? { ok: true, method, name, value: `property-proof=${token}` }
    : {
        ok: true,
        method,
        name,
        value: `${token}.proof.invalid`,
      };
}

const existing: DnsRecord[] = [
  {
    name: "_dmarc.harborview.example",
    type: "TXT",
    values: ["v=DMARC1; p=none; rua=mailto:dmarc@harborview.example"],
  },
];

console.log(planDomainProof("harborview.example", "tnt_7f3a91", existing, "TXT"));
```

The `.invalid` suffix is intentional in sample data: it cannot accidentally direct a reader toward a real service. Production code would use an application-controlled target and a cryptographically random token. The verifier should compare a parsed DNS answer, not substring-match a raw response, and it should scope every token to the expected tenant and domain.

Do not turn a negative lookup into an immediate customer failure. DNS answers are cached, and an onboarding UI should preserve the pending state, show the exact expected owner name and value, and let the customer retry. On success, store enough evidence to explain the decision later: tenant, requested domain, method, expected token identifier, observed record, verification time, and verifier version. Avoid storing a reusable secret when a hash or non-secret token identifier is enough for the audit record.

This is the expensive failure.

## Ownership proof is not deliverability evidence

For property communications, the damaging mistake is letting a verified-domain badge imply that mail is ready. The proof only establishes that someone could publish the requested DNS value. It says nothing by itself about DMARC policy, identifier alignment, message signing, recipient acceptance, or whether the sending system uses the intended domain.

DMARC evaluates mail using identifiers associated with SPF and DKIM and applies alignment rules described by the standard. Its policy record also supports aggregate reporting. Those reports are valuable evidence over time because they can reveal which sources send using the domain and whether authentication aligns. They do not promise inbox placement, and the absence of a report is not proof that mail was delivered.

I would expose two states in the product model:

- `control`: pending, verified, expired, or revoked
- `mailEvidence`: not_checked, incomplete, observable, or policy_enforced

The exact labels can change. The separation should not. A leasing team can finish domain ownership verification while an administrator completes mail configuration, and support can see which track is blocked without asking the customer to paste screenshots from a DNS dashboard.

One badge is not enough.

A useful evidence view records what was actually observed. For example, show that the ownership token matched at 14:32 UTC, that a DMARC record was present on the same evaluation, and that reporting destinations were syntactically parsed. Do not translate those observations into a guarantee of delivery. Recipient systems make their own decisions, and DMARC itself is only one part of mail authentication and handling.

## Choose by authority boundary, not by familiarity

TXT and CNAME proofs answer slightly different operational questions. A TXT token asks, “Can this customer publish a precise value under this domain?” A CNAME asks, “Can this customer delegate this dedicated hostname to a target we operate?” For basic ownership, the first question is enough. For a long-lived hostname whose destination the application must update, delegation may be the intended contract.

The trade-off is easy to state and easy to ignore. TXT minimizes the DNS surface controlled by the application, but rotating or rechecking the proof requires another customer-side edit unless the original proof remains acceptable. CNAME lets the application change the destination behind its own target, but it consumes the entire owner name and creates a continuing dependency on that target. Deleting a tenant must therefore include retiring the target and preventing stale aliases from being claimed by another tenant.

**Default to TXT for proof; require a dedicated CNAME only for delegation.** This is not a claim that TXT is universally better. It is a narrower decision rule that matches the permission being requested.

Cost enters indirectly. DNS queries and small records are rarely the expensive part of an LLM-backed property product. Human support is. The cheapest design is the one that prevents an administrator from receiving impossible instructions, gives support an evidence trail, and does not force a founder to debug an ambiguous “domain failed” state while also operating the application. That is why I would spend engineering effort on validation and state modeling before polishing provider-specific setup screens.

## Operate the proof as a lifecycle

Start by normalizing the requested domain and binding it to exactly one tenant. Generate a fresh token, select the method from the authority boundary, and check the proposed owner name for incompatible data before presenting instructions. The UI should display the fully qualified name, record type, exact value, and current state. Keep mail-readiness language off this screen unless it is backed by a separate evaluation.

During verification, query through the same resolver strategy used by the production verifier and retain the parsed observation. Pending should remain pending until the expected value is visible; malformed or conflicting records deserve a specific result. Once control is established, evaluate the mail evidence independently. For DMARC, observe the policy record at the standard location, parse it, and surface what it says rather than inventing a delivery score.

Then plan for change. Recheck control when a sensitive domain setting changes, expire abandoned challenges, and revoke the binding when the tenant disconnects the domain. For CNAME delegation, also monitor the target lifecycle so a dangling customer alias cannot outlive its tenant mapping. For TXT, decide whether the proof record may be removed after verification and make that contract clear; if continuous proof is required, say so before the customer cleans up DNS.

Finally, test the awkward states: TXT at the proposed CNAME name, CNAME at the proposed TXT name, multiple TXT values, mixed letter case, a trailing dot, a delayed answer, token replay against another tenant, and deletion followed by reconnection. These cases are more useful than another happy-path screenshot. They expose whether the system has a real domain-control model or only a string comparison attached to a spinner.

The implementation can stay small. The contract cannot be vague. Keep proof and deliverability separate, reserve owner names deliberately, and retain evidence that explains every state transition.

## Further reading and References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
