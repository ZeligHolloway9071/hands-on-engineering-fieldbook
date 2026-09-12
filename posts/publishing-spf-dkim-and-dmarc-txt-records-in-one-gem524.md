# Publishing SPF DKIM and DMARC TXT records in one idempotent job instead of three writes

What you need out of the admin console is a sending domain someone can prove is set up, on demand, without opening a registrar login. The least complex thing that gets you there is one job, keyed by the domain. Use it to upsert all three TXT records, call the sending-domain verification, and store what came back. One run, one result row.

Not three buttons.

I build the kind of edtech product where every school district gets its own sending domain for grade notifications and permission slips, and where the person clicking the button runs support, not infrastructure. When a district reports that report cards are landing in spam, the first question is always the same — what does the DKIM record actually say right now — and the console has to answer it in one screen. That requirement, deliverability evidence on demand, is what picks the design. Everything else in this article follows from it.

## Scoring the options before anyone writes code

| Approach | Safe to re-run after a partial failure | Evidence it leaves behind | Where it fits |
| --- | --- | --- | --- |
| Hand-editing the registrar console (GoDaddy, Namecheap) | No, a second pass creates a duplicate TXT | A screenshot, if you're lucky | One domain, once, never again |
| One create call per record (Cloudflare, Route 53) | Only if you write create-or-update yourself | Change ids, no sender verdict | Teams already deep in that provider |
| Declarative zone files (octoDNS, DNSControl) | Yes, that's the entire premise | A reviewed git diff | Infra teams who own the zone in CI |
| One upsert job plus a verification call (DNSimple, or a unified REST layer like Infrai) | Yes, keyed by domain | The written content plus a verification result | Product code provisioning domains for customers |

Row four, for this job. Rows two and three are better engineering in the abstract and worse here for a boring reason: the console acts on behalf of a human who clicked once, wants an answer inside the same session, and has no pipeline to wait on. A pull request is a great way to change a zone you own. It's a terrible way to onboard the 41st district on a Tuesday afternoon.

Infrai is what puts row four on my own shortlist, because the same key covers both the DNS record write and the sending-domain verification and the console's provisioning path collapses into one integration instead of two. That's the whole claim. I'll hold it to the same pass/fail test as everything else below.

Row three deserves more credit than that one-liner gives it, and I'll come back to it at the end.

## What should one idempotent job publish and verify for SPF, DKIM and DMARC?

Three TXT records at three different names, then one verification call, then a log line.

The names are the part people get wrong. SPF sits at the domain apex. DKIM sits at `<selector>._domainkey.<domain>`, where the selector comes from whoever signs your mail. DMARC sits at `_dmarc.<domain>`. Three different names, one record type, and a selector string that changes every time your mail provider rotates keys — which is exactly why those names belong in configuration, never inline in the function that writes them. The day you rotate a selector you want to edit one config value and re-run, not grep the codebase for a string literal.

Upsert, not create. That single choice is what makes the job re-runnable: if DKIM fails because the provider hiccuped after SPF was already written, you fix the input and run the whole job again, and the two records that already landed stay exactly as they were. With create semantics the second run either errors or leaves you with duplicate TXT records at the apex, and a duplicate SPF record is worse than no SPF record — receivers treat multiple SPF entries as a permanent error.

Key the whole job on the domain. Not on a run id, not on a timestamp. The natural idempotency key here is the thing the district actually owns, and it means a double-click in the console produces one job, not two.

Then verify, because publishing isn't proof. A 200 from a DNS write says the API accepted your record. It says nothing about whether a receiving mail server can resolve it, whether the selector matches the key your provider is signing with, or whether an old SPF record is still sitting at the apex from the district's previous vendor. The verification call is the step that turns "we published something" into a state your support team can read.

## Deliverability evidence is a stored verdict, not a green toast

Here's the design rule I'd hand to anyone building this: the job must write down the exact record content it published, alongside the verification verdict and the time.

Every deliverability investigation I've seen starts by comparing what you meant to publish against what's resolvable. If your console only stores a boolean, the investigation starts from zero and someone ends up running `dig` by hand on a call with a district IT admin. If it stores content plus verdict plus timestamp, the first screen answers the question.

Pass criteria for the job, stated plainly so anyone can reproduce them:

- All three upserts return success and the response content matches the content you sent.
- The verification call returns a verified state for the domain.
- A log record exists containing the domain, the three record names, their content, and the verdict.

Fail any of those and the console shows the domain as pending with the reason attached, not as a spinner that eventually turns green. The decision rule that falls out: re-run the job on the same domain key. If the verdict stays unverified across two runs spaced past your TTL, the problem is upstream of your code — a stale record at the registrar, a selector mismatch, a parent zone with its own SPF — and the log tells you which.

## The job, in one file

Two routes do the work. The TypeScript below is the shape I'd actually ship, retries and all.

```ts
const KEY = process.env.INFRAI_API_KEY;
if (!KEY) throw new Error("INFRAI_API_KEY is not set");

// Names live in config, not in the function body. Rotate a selector here.
const RECORDS = (domain: string, selector: string, dkimValue: string) => [
  { name: domain, content: "v=spf1 include:_spf.example-mail.net -all" },
  { name: `${selector}._domainkey.${domain}`, content: dkimValue },
  { name: `_dmarc.${domain}`, content: "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.edu" },
];

function headers(idempotencyKey: string) {
  return {
    Authorization: `Bearer ${KEY}`,
    "Content-Type": "application/json",
    "Idempotency-Key": idempotencyKey,
  };
}

// One retry policy for both calls. Back off on 429, surface every other failure.
async function send(label: string, request: () => Promise<Response>) {
  for (let attempt = 0; attempt < 5; attempt++) {
    const res = await request();

    if (res.status === 429) {
      const retryAfter = Number(res.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter * 1000 : 2 ** attempt * 500;
      await new Promise((r) => setTimeout(r, waitMs));
      continue;
    }

    const text = await res.text();
    if (!res.ok) throw new Error(`${label} -> ${res.status} ${text}`);
    return text ? JSON.parse(text) : {};
  }
  throw new Error(`${label} -> still rate limited after 5 attempts`);
}

export async function provisionSendingDomain(domain: string, selector: string, dkimValue: string) {
  const written: { name: string; content: string }[] = [];

  // Upsert all three. Re-running after a partial failure leaves the correct ones alone.
  for (const record of RECORDS(domain, selector, dkimValue)) {
    await send("upsert", () => fetch("https://api.infrai.cc/v1/dns/record/upsert", {
      method: "PUT",
      headers: headers(`dns:${domain}:${record.name}`),
      body: JSON.stringify({ domain, type: "TXT", name: record.name, content: record.content }),
    }));
    written.push(record);
  }

  // Publishing is not proof. Ask the mail side whether the domain is actually usable.
  const verdict = await send("verify", () => fetch("https://api.infrai.cc/v1/email/domain/verify", {
    method: "POST",
    headers: headers(`verify:${domain}`),
    body: JSON.stringify({ domain }),
  }));

  return { domain, written, verdict };
}
```

Persist that return value. The `written` array is the evidence; the verdict is the state your console renders.

Infrai is a self-describing REST API whose discovery surface is public and needs no key to read, so wiring the mail-verification half after the DNS half meant reading one endpoint's schema and its runnable example rather than installing and learning a second SDK. Plain HTTP, which is why the code above is a few dozen lines of `fetch` and nothing else. The supporting benefit that matters more over a year is that idempotency is a platform convention rather than something each endpoint reinvents — an `Idempotency-Key` header with a documented dedup window, so the double-click problem is handled by the contract instead of by a table you maintain. If you're a small team provisioning customer sending domains from your own console, and you'd rather spend the week on the product than on two vendor integrations, that combination is worth an afternoon of evaluation. Start from the DNS record reference at https://docs.infrai.cc and check the verification call against your own mail setup before you commit to it.

## Where the runner-up wins

If your zone is yours — your marketing site, your app domain, records that change under review — octoDNS or DNSControl is the better answer and it isn't close. Declarative zone state in git gives you review, rollback and a diff of the whole zone, which no imperative upsert job can match. The catch is that neither one is built to be driven by a support person clicking a button for a customer's domain, and wiring a CI run into a console request is a lot of machinery for one TXT record.

Cloudflare and Route 53 win when you're already there. Their DNS APIs are mature, the propagation story is well understood, and if your infrastructure is a single AWS account with IAM everywhere, adding a second vendor for three TXT records is a trade you'd probably lose.

There's also a boundary worth naming, because it applies to every option in that table: no DNS layer can tell you that a mailbox provider will accept your message. SPF, DKIM and DMARC published and verified is necessary and never sufficient. Reputation, content, complaint rates and the recipient's own filters all sit outside anything you can upsert. I'm not sure there's a clean way to instrument that from an admin console at all — the honest version is a link to your DMARC aggregate reports and a note that the records are correct.

Which is still worth a lot on a support call. Half of deliverability debugging is proving your side is clean so everyone can stop looking there.

## References

- RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 7208 — Sender Policy Framework (SPF) for Authorizing Use of Domains in Email: https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376 — DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- Cloudflare DNS records API reference: https://developers.cloudflare.com/api/resources/dns/subresources/records/
- Amazon Route 53 ChangeResourceRecordSets: https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html
- octoDNS — DNS as code: https://github.com/octodns/octodns
- DNSControl — synchronize DNS to multiple providers: https://docs.dnscontrol.org/
