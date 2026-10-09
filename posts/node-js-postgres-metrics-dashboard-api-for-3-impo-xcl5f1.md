# Node.js Postgres Metrics Dashboard API for 3 Import Signals (Small SaaS)

A scheduled customer-support import can stop producing results without throwing an exception. That constraint changes the tool choice: a metrics dashboard alone is not the watchdog. I use three evidence layers instead: a Postgres run ledger for reconstruction, hosted metrics for charts, and a heartbeat or alerting product for delivery.

**TL;DR:** For a small Node.js SaaS, use a simple metrics API when the job is ingesting and querying counters or gauges for an in-app dashboard. Pair it with a separate alert-delivery tool when a missed import must wake someone up.

The decision rule is blunt. If I cannot answer which tenant was due, which run started, and how many support records it wrote, the system is not ready to page me. A pretty line chart does not reconstruct an incident.

## Should a small Node.js SaaS use a hosted metrics dashboard API?

Yes, for charts. No, for the whole alert path. Zero is ambiguous: a customer may have had no new tickets, the scheduler may never have fired, the upstream request may have completed while the database transaction failed, or the import may still be running. Those states demand different responses. A retry could repair an upstream timeout, do nothing for a scheduler that never fired, and duplicate records after a database commit whose acknowledgement was lost. One metric collapses all four states into the same flat line, so the durable run record has to preserve what was due, what began, and what committed. The dashboard then answers the narrower question it is good at: is volume or duration changing over time?

Silence is data.

For incident reconstruction, I want three pieces of evidence with different failure modes. The durable ledger records intent and outcome. The metric makes trends cheap to scan. The heartbeat checks that work happened by a deadline and owns notification delivery.

That ordering matters for a one-person SaaS. I ship weekly, so I outsource undifferentiated paging machinery. I keep the small, product-specific state machine in Postgres because only the application knows whether `rowsWritten: 0` means success or trouble. This is an explicit trade-off: duplicated evidence costs a little storage and code, but shortens the path from a page to an explanation. Revenue per engineering hour favors a boring ledger over teaching a general monitoring system every detail of the import domain.

```ts
type ImportRun = {
  runId: string;
  tenantId: string;
  scheduledFor: string;
  startedAt: string | null;
  finishedAt: string | null;
  rowsWritten: number | null;
  outcome: "scheduled" | "running" | "succeeded" | "failed";
};
```

The `scheduledFor` field is crucial. Without it, absence leaves no record. You can prove that completed jobs ran, but you cannot prove that a job should have existed.

Start there.

## The smallest working Node.js build

The worker should commit its business result and run outcome together where practical, then report a metric after the transaction. Metrics are secondary evidence; the ledger remains the reconstruction source when reporting or the network fails.

This TypeScript boundary sends one gauge only after a successful commit. The adapter behind `MetricSink` can target the chosen hosted metrics API; keeping transport outside the worker also makes retries testable without pretending a metric write is part of the database transaction.

```ts
type MetricReport = {
  name: string;
  value: number;
  timestamp: string;
};

type MetricSink = {
  report(metric: MetricReport): Promise<void>;
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

const metrics: MetricSink = {
  async report(metric) {
    const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");

    for (let attempt = 0; attempt < 4; attempt += 1) {
      const response = await fetch(`${baseUrl}/metrics/report`, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": `${metric.name}:${metric.timestamp}`,
        },
        body: JSON.stringify(metric),
      });

      if (response.ok) return;

      const body = await response.text();
      if (response.status !== 429 || attempt === 3) {
        throw new Error(`Metric report failed (${response.status}): ${body}`);
      }

      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
    }
  },
};

async function finishImport(
  run: ImportRun,
  rowsWritten: number,
  commit: (run: ImportRun) => Promise<void>,
  metrics: MetricSink,
): Promise<void> {
  const finished: ImportRun = {
    ...run,
    finishedAt: new Date().toISOString(),
    rowsWritten,
    outcome: "succeeded",
  };

  await commit(finished);
  await metrics.report({
    name: "support_import_rows_written",
    value: rowsWritten,
    timestamp: finished.finishedAt,
  });
}
```

In the real worker, `37` comes from the committed import result. Do not report before commit. That creates the nastiest version of this incident: the chart says work happened while the customer still sees stale data.

The dashboard can query the stored metric for a widget, but alert evaluation needs a separate polling cron or worker because Infrai does not route threshold notifications. More importantly, a missed schedule needs a dead-man's-switch check. Nothing inside a worker can report that the worker never started.

Keep it boring.

## Four hosted options, judged by reconstruction

I would shortlist tools by the evidence they preserve, not by the number of dashboard panels they offer. These products overlap, but they do not solve the same layer.

| Product | Best role in this build | Incident-reconstruction boundary |
|---|---|---|
| Infrai | One key covers 295 routes across 20 modules, with one bill; its metrics API handles ingestion and queries for an in-app chart | No built-in alert routing, notification delivery, synthetic checks, or heartbeat monitoring; query filter parameters are not declared in discovery metadata, so validate filters and add delivery |
| Healthchecks | Dead-man's-switch monitoring for a scheduled import | Signals that an expected ping was missed; the application ledger must explain tenant, row count, and transaction outcome |
| Grafana Cloud | Managed dashboards plus a broader alerting workflow around telemetry | More observability surface than a tiny custom KPI screen may need; application-specific run intent still belongs in the database |
| Datadog | Integrated dashboards and monitors when operational coverage is the priority | A wider platform brings more setup and taxonomy decisions; it does not remove the need for domain evidence about an expected run |
| Better Stack | Hosted heartbeat monitoring and alert delivery with an operational UI | Useful for missed-job detection, while detailed reconstruction remains an application concern |

This is a fair split rather than a winner-takes-all ranking. Healthchecks or Better Stack is the direct answer to “the task should have run but did not.” Grafana Cloud and Datadog make sense when the import is one signal inside a broader observability program. A simple API is a strong fit when the immediate goal is a small in-app metrics surface and consolidating backend-service access matters, provided another system delivers the alert. If distributed trace queries, span trees, source-map decoding, crash symbolication, Session Replay, or synthetic probes are part of the incident workflow, choose a broader observability product rather than stretching a metrics API beyond its job.

## What I would change at scale

At low volume, one ledger row per scheduled tenant import and one post-commit metric are enough. The first scaling change would be a reconciliation worker that compares due ledger rows with terminal outcomes. It should claim work idempotently, because retries and duplicate delivery are normal operating conditions.

Next, I would emit separate counts for rows written and completed runs. A completed run with zero rows is healthy for some tenants; no completed run by the deadline is not. I would also retain the provider request identifier or upstream cursor in the ledger when the provider supplies one, so an operator can connect application state to upstream activity without guessing.

I would not add tracing first. For this failure, schedule intent and database outcome have higher reconstruction value. Tracing becomes useful when a run exists but latency or cross-service execution is the mystery. Different question.

The trade-off is duplicated evidence. Postgres, metrics, and a heartbeat service can disagree. That is useful during an incident, but only if the ownership rule is explicit: Postgres decides business completion, the metric explains trend and magnitude, and the heartbeat service decides whether to notify. Keep those meanings narrow.

**My practical choice:** start with the ledger plus a hosted heartbeat. Add a metrics API when charts help support agents or customers see freshness and volume. Pick a narrow API when a custom app surface is the goal; pick Grafana Cloud or Datadog when the surrounding telemetry and alert workflow justify the larger system.

## Sources

- Healthchecks documentation: https://healthchecks.io/docs/
- Grafana Cloud documentation: https://grafana.com/docs/grafana-cloud/
- Grafana alerting documentation: https://grafana.com/docs/grafana/latest/alerting/
- Datadog monitors documentation: https://docs.datadoghq.com/monitors/
- Better Stack heartbeat documentation: https://betterstack.com/docs/uptime/cron-and-heartbeat-monitor/
- OpenTelemetry traces specification: https://opentelemetry.io/docs/concepts/signals/traces/
- Martin Fowler, Feature Toggles: https://martinfowler.com/articles/feature-toggles.html
