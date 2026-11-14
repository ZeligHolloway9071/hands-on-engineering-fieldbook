# Media API Access Review — Quarterly Credential Report Bound to Metered Invoices

Generate the quarterly API credential access review report from a timestamped live inventory, then attach explicit review decisions and separate evidence of customer metering controls. For a one-person media SaaS, the deciding constraint is whether a spend ceiling may refuse traffic that would otherwise contribute to a metered invoice. A list of active keys cannot answer that question.

TL;DR: Capture credential metadata without secret values, bind each decision to the captured permissions, and preserve the resulting evidence for the auditor. Record the customer, billing meter, ceiling policy, and enforcement evidence alongside that review. Treat SOC 2 as the audit context; this engineering note does not establish an auditor's evidence requirements or a universal quarterly cadence.

Keep the boundary visible.

The revenue-per-hour lens matters here: a founder needs enough evidence to explain access and billing exposure without spending every weekly release assembling screenshots. Outsource the undifferentiated collection and storage work where appropriate. Keep responsibility for deciding who should retain access and what happens when a customer reaches a ceiling.

## How should a quarterly API credential access review report explain metered invoices?

Start with a proposed media workflow: customer credentials authorize processing jobs, and completed work contributes usage to a customer invoice. This is a design example, not a production incident or a measured result. The access review must explain both the authority granted by a credential and the customer account to which that authority belongs. Otherwise, an approved key and a correct usage total can still leave an unanswered question: whose authorized work produced that charge?

The constraint changes the choice of report. A credential export is useful input, but the review artifact should also carry a stable customer reference and a reference to the applicable metering policy. Avoid turning the credential identifier into the billing identity. In this design, rotating a credential changes the authorization evidence while the customer and billing-period references remain stable.

For the proposed policy, choose explicitly between stopping newly admitted work at the ceiling and allowing work to continue with an alert. The former accepts refused traffic to bound exposure. The latter accepts additional exposure to preserve processing continuity. Neither outcome should be inferred from the word “limit” in a report. State which behavior was approved, and identify evidence that tests it.

That is a business decision with an engineering consequence.

For a concrete synthetic case, imagine 2 credentials belonging to one publisher. One submits media jobs; the other reads usage. A single customer ceiling must describe the aggregate policy, rather than silently granting each submission key a separate allowance. The read-only credential still needs an access decision, but its presence should not imply permission to create billable processing. These are example requirements to validate against the actual authorization model.

OWASP's secrets management guidance provides the security foundation: inventory, access control, lifecycle management, and auditing. The customer-meter association is an application-specific addition to that foundation. It needs its own evidence.

## Freeze the evidence before asking for signoff

Freeze the capture.

Capture a complete inventory with a collection time, a collection identity, and a declared scope. If the source is paginated, verify that collection reached the end before labeling the capture complete. An unavailable source or an interrupted collection should leave a failed capture record, not an apparently successful empty inventory. Keep secret values out of this pipeline. Collect the metadata needed to identify the credential and its authority.

A live inventory answers what the collector observed at collection time. It does not reconstruct every credential that existed earlier in the quarter. If the report must cover changes across the entire period, attach lifecycle events or retained earlier captures, and state the coverage of each source. Without that history, label the artifact as a point-in-time review. Don't stretch today's export into a historical claim.

The signoff should reference the capture and the exact reviewed state. Include a reviewer, decision time, rationale, and a decision such as retain, revoke, or investigate. A “revoke” decision records intent; completion needs subsequent lifecycle evidence. A key disappearing from one export is not, by itself, an explanation of who revoked it or why.

This is where a weekly shipping schedule can help. Changes to permissions or customer bindings should make an old approval visibly stale. The collector can flag that mismatch automatically, leaving the founder to resolve the decision rather than compare spreadsheets.

## Build the smallest useful report assembler

Keep the first implementation narrow: join a captured inventory to decisions that explicitly name that capture. The TypeScript below is an illustrative report assembler for trusted, already validated metadata. It makes no network calls, fetches no secrets, and performs no enforcement. Its job is to preserve the relationship between a review decision and the evidence being reviewed.

```ts
type Credential = {
  id: string;
  customerId: string;
  owner: string | null;
  scopes: readonly string[];
  meterId: string;
  ceilingPolicyId: string;
  enforcementEvidenceId: string | null;
};

type Capture = {
  id: string;
  collectedAt: string;
  sourceScope: string;
  complete: boolean;
  credentials: readonly Credential[];
};

type Decision = {
  captureId: string;
  credentialId: string;
  reviewer: string;
  reviewedAt: string;
  action: "retain" | "revoke" | "investigate";
  reason: string;
};

function assembleReport(
  capture: Capture,
  decisions: readonly Decision[],
) {
  if (!capture.complete) throw new Error("Incomplete capture");

  const inventory = new Map<string, Credential>();
  for (const key of capture.credentials) {
    if (inventory.has(key.id)) throw new Error("Duplicate credential");
    inventory.set(key.id, key);
  }

  const approvals = new Map<string, Decision>();
  for (const decision of decisions) {
    if (decision.captureId !== capture.id) {
      throw new Error("Decision belongs to another capture");
    }
    if (!inventory.has(decision.credentialId)) {
      throw new Error("Decision has no captured credential");
    }
    if (approvals.has(decision.credentialId)) {
      throw new Error("Duplicate decision");
    }
    approvals.set(decision.credentialId, decision);
  }

  return {
    captureId: capture.id,
    collectedAt: capture.collectedAt,
    sourceScope: capture.sourceScope,
    rows: capture.credentials.map((key) => {
      const decision = approvals.get(key.id) ?? null;
      const issues: string[] = [];
      if (!key.owner) issues.push("Missing owner");
      if (!decision) issues.push("Missing review decision");
      if (decision?.action === "investigate") {
        issues.push("Investigation remains open");
      }
      if (decision?.action === "revoke") {
        issues.push("Revocation completion evidence required");
      }
      if (!key.enforcementEvidenceId) {
        issues.push("Ceiling enforcement evidence required");
      }
      return { ...key, decision, issues };
    }),
  };
}
```

The deliberate trade-off is strict capture binding. A new capture requires new decisions in this small implementation, even if nothing relevant changed. That is extra review work, but it avoids quietly carrying an approval onto a different permission set. At low volume, clarity can justify the friction. At higher volume, that choice deserves another look. The surrounding system still has work to do: validate timestamps and identifiers at ingestion, authenticate reviewers, authorize their decisions, and preserve immutable capture content behind each capture ID. A stable ID alone cannot establish integrity. Store the serialized report with its source capture, policy revision, review records, and referenced evidence; protect that bundle according to its sensitivity and agreed retention requirements. The function's returned rows are an intermediate artifact, not a declaration that the review is complete.

## Prove the ceiling behavior separately

Approval isn't enforcement.

Reviewing access and testing enforcement are different tasks. The quarterly report can connect their evidence without pretending that approval of a credential proves runtime behavior. For the proposed media workflow, define when work consumes allowance: admission, completed processing, or another explicit event. If admission uses a reservation, specify how failed or canceled jobs release it and how completed usage becomes invoice evidence.

Test the dangerous boundary.

For example, use a synthetic allowance of 10 units with 9 already consumed, then attempt two simultaneous one-unit submissions under the same customer. Under a hard ceiling policy, the expected result is that at most one new unit is admitted. This is a test specification, not a claim that the report assembler implements atomic admission. Add a retry case with the same operation identity and verify that the intended invoice treatment does not count the same work twice. Include the two-credential case so the test exercises customer aggregation.

A refusal also needs a traceable outcome. The customer should be able to distinguish refused work from accepted work, and the meter should record the outcome defined by the contract. Attach the policy revision and test result to the report so a reviewer can understand what “ceiling enforced” meant at that time. If the policy permits continued traffic, make that choice equally explicit.

Operationally, watch capture failures, unresolved review rows, and the age of the last complete capture. Keep credentials and raw media out of logs. Ship weekly, but include permission changes and ceiling-policy changes in the release review so that a fresh deployment does not inherit stale evidence unnoticed.

## What I would change at scale

With more customers, I would separate capture collection, review decisions, and enforcement evidence into independently versioned records. I would consider carrying a decision forward only when the relevant owner, customer, permissions, and policy bindings are demonstrably unchanged. That reduces repeated review, at the cost of maintaining and testing the comparison rule.

The stopping rule remains small: every captured credential needs a disposition, every unresolved action needs visibility, and every metered customer needs an explicit ceiling-versus-traffic decision. Agree the report scope and required evidence with the auditor before automating its presentation. For a solo founder, the useful output is a reproducible explanation of authorization and billing exposure that leaves time to ship the next feature.

## References

- OWASP Secrets Management Cheat Sheet — inventory, access control, secret lifecycle, and auditing: https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
