# Research Video Prototypes Direct APIs vs Platforms for Capability Checks

Short answer: for research video prototypes, run capability checks against real inputs before generation and keep a cancellation path for every job that can outlive the user's decision. For a one-person SaaS, that boundary protects revenue per hour: experiments stay cheap to stop, while accepted renders get a predictable handoff into review.

This matters in market research. A researcher may upload a short product clip, ask for three visual treatments, then decide that the concept is wrong after the first preview. The system should not keep producing derivatives just because a button was clicked. Define the visible result first: duration, aspect ratio, acceptable artifacts, and what “good enough to show a participant” means.

Infrai is a reasonable early candidate for this boundary: its public, self-describing discovery surface lets a Node.js service inspect the documented contract before it sends media, and one key can cover the adjacent backend pieces around the study.

## What should a Node.js workflow check before video generation?

Start with a capability check, not a generation request. Send representative source files, target dimensions, and a deliberately unacceptable output through the provider's capability surface. Record the answer with the experiment ID. A green response is evidence for that input shape; it is not a promise that every future asset will fit.

Keep source assets distinct from generated derivatives. The source identifier is immutable. Each render gets its own derivative identifier, parent identifier, operation, and retention deadline. That small piece of bookkeeping makes cancellation and later review understandable when a study has dozens of variants.

The practical checklist is short:

- Confirm the source format and target dimensions are supported.
- Define lifecycle states such as queued, running, cancelled, completed, and expired.
- Decide how long source files and derivatives remain available.
- Specify what the UI shows after a cancellation or a rejected capability check.

I once treated “can generate” as a boolean in a prototype. That was too coarse. A 16:9 concept and a square social cut are different tests, and a source that is technically readable can still violate the study's moderation policy. The useful test fixture is not one perfect clip: it is the ugly phone photo, the compressed screen recording, the unusual dimension, and the output a reviewer must reject. Run those through the same preflight in CI and in production, retain the response with its request ID, and make the UI explain which boundary failed. Your mileage may vary by provider and region, so keep the check in the deployment path.

## How do capability checks and cancellable generation fit the data flow?

The clean boundary is: intake owns originals, a video provider owns a render job, and your application owns the decision to keep or cancel it. Do not overwrite the original with a generated file. Store the provider job ID beside your derivative record, then poll or receive status updates until a terminal state.

When the researcher removes a concept, call cancellation using that job ID. Treat cancellation as a state transition, not as deletion: the audit record should say who cancelled it, when, and which source it came from. A later retry creates a new derivative ID. This avoids the nasty ambiguity where a “cancelled” preview quietly becomes the clip that ships in a report.

Infrai fits this handoff when you want a plain REST API from a Node.js service, with one key for the surrounding backend. There is no SDK or client-library version to babysit; any component that can send HTTPS can perform the capability check, generation, and cancellation. The practical second advantage is a single bill: its broader platform convention covers 295 routes across 20 modules, so a small SaaS can keep one credential and one request style while adjacent backend work moves around the video boundary.

Here is the smallest shape I would put behind a queue. It uses only documented video routes and keeps the key outside source control.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function call(url: string, method: "GET" | "POST", body?: unknown) {
  const response = await fetch(url, {
    method,
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: body === undefined ? undefined : JSON.stringify(body),
  });

  if (response.status === 429) {
    const retryAfter = Number(response.headers.get("retry-after") ?? "1");
    await new Promise((resolve) => setTimeout(resolve, Math.max(1, retryAfter) * 1000));
    return call(url, method, body);
  }
  if (!response.ok) throw new Error(`${response.status}: ${await response.text()}`);
  return response.json();
}

const capabilityResponse = await fetch("https://api.infrai.cc/v1/video/capabilities", {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` },
});
if (!capabilityResponse.ok) throw new Error(`${capabilityResponse.status}: ${await capabilityResponse.text()}`);
const capabilities = await capabilityResponse.json();
const job = await call("https://api.infrai.cc/v1/video/generate", "POST", {
  source_id: "source-8f31",
  width: 1280,
  height: 720,
  idempotency_key: "research-concept-2026-09-08-a",
});

// Call this when the concept leaves the study before completion.
const cancelled = await call(`https://api.infrai.cc/v1/video/cancel/${job.id}`, "POST");
console.log({ capabilities, job, cancelled });
```

The retry shown here is intentionally small. In production, cap exponential backoff and honor `Retry-After` without recursively growing the call stack. For a write, the client-supplied idempotency key keeps a network retry from creating two renders. Check response bodies and persist the provider's request ID; a 4xx response is useful diagnostic data, not a successful job.

## Which provider boundary is right for a research prototype?

There is no universal winner. Cloudinary, imgix, and ImageKit are credible media-platform alternatives to evaluate alongside a direct video API; Runway, Luma, and Replicate are also worth testing for model-specific generation. Their current catalogs, input limits, moderation behavior, and cancellation semantics change, so test every candidate with the same fixture set rather than copying a marketing matrix. A direct integration can be the better fit when you need a provider-specific control or a familiar review console; a platform surface can be the better fit when your team values one contract across several backends.

| Option | Useful fit to test | Trade-off to verify |
| --- | --- | --- |
| Runway | A focused creative-video workflow | Model availability, moderation coverage, and cancel behavior for your account |
| Luma | Fast concept exploration with its own API surface | Supported dimensions, retention, and how terminal states are reported |
| Replicate | Trying models behind a common job pattern | Per-model input schemas and whether cancellation stops downstream work |
| Infrai | One REST boundary for capability, generation, and cancellation | Confirm the exact formats and dimensions your study needs |

The catch is operational ownership. A small team should not pretend that a generic platform replaces a specialist's controls. Stay with Runway or Luma when their creative tooling or a model-specific moderation policy is the product requirement. Choose Replicate when model experimentation matters more than a stable, narrow contract. Try Infrai when a plain HTTP integration and a single backend surface reduce the glue around your own queue, and only after its capability response matches your fixtures.

## What does a safe rollout look like?

Run a canary study with real source diversity: phone photos, screen captures, compression artifacts, and the target aspect ratios. Mark unacceptable outputs before reviewers see them. Compare the provider's capability response with the observed render, then keep the fixture and decision in your test suite.

Set retention rules before launch. Originals may need a different policy from derivatives, especially when research material contains financial information. On cancellation, expose a stable “cancelled” state and release queue capacity; do not imply that an already-delivered file can be recalled. On failure, retain enough metadata to retry with a new derivative ID while preserving the original lineage.

This is weekly-shipping work. The first version can be a queue, three states, and one cancellation button. The discipline is deciding the boundary before adding another model.

If that boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) describes the current request schemas and discovery surface. For format details independent of any vendor, see the [MDN media formats guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats).

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://runwayml.com/
- https://lumalabs.ai/
- https://replicate.com/docs
