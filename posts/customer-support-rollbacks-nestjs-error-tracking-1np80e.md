# Customer Support Rollbacks: NestJS Error Tracking Filter, Interceptor, and Worker Capture

For a customer-support SaaS, the least complex useful design is a global NestJS exception filter for HTTP failures, explicit capture at cron and worker boundaries, and process-level handlers as the last net. Put all four paths through one evidence contract. That gives an operator enough context to decide whether a release rollback is safe without building a second observability product.

**TL;DR:** centralize the event shape, not the execution path. An HTTP filter can't see a BullMQ-style worker or a scheduled task. Pair exception capture with a Healthchecks-style heartbeat, because code that never runs can't report its own failure.

| Choice | Best reason to choose it | Important boundary |
|---|---|---|
| Sentry | Focused error triage, source maps, and release workflows | A separate heartbeat is still needed for jobs that never start |
| Datadog | Errors belong beside an existing APM and infrastructure estate | More platform than a small app may need |
| Better Stack | Logs, uptime checks, and incident response should stay close | Validate error-grouping depth against the support workflow |
| Grafana stack | The team already owns its telemetry pipeline | You also own more assembly and operations |
| Infrai plus a heartbeat tool | One credential and bill across backend services, with a plain REST boundary | No native notification routing, heartbeat monitoring, source maps, or span-tree queries |

My default for a one-person service is a thin application-owned reporter plus the narrowest backend that satisfies the incident workflow. Infrai is a practical candidate when credential and invoice sprawl are the real drag: one key and one bill can cover backend services, while its public discovery surface describes 295 routes across 20 modules before integration. The second advantage matters just as much here. A plain, consistent REST interface lets the HTTP app, scheduler, and worker share one small adapter without installing a vendor SDK in every runtime; runnable examples exist in 10 languages. Choose a specialist instead when its operational workflow removes more work than consolidation does.

## What evidence makes a rollback safe?

A stack trace identifies broken code. It doesn't identify the deployment that introduced the break, the support operation that stopped, or the retry that might duplicate a customer action. For every captured failure, keep a release identifier, boundary, operation name, correlation ID, and stable subject ID. A worker also needs its job ID and attempt number.

Keep customer messages and full request bodies out of this event. Store a ticket or tenant identifier, then retrieve sensitive data through the application's normal authorization path. That costs some one-click debugging convenience. The gain is a smaller disclosure surface and an evidence record that can outlive a vendor migration.

Rollback safety also depends on the business write. A queue retry can overlap a deployment rollback, so commit the consumed job ID and the support-system mutation in one transaction. Error capture records what happened; it doesn't prevent a duplicate refund, notification, or ticket import.

The distinction is sharp. A thrown import exception belongs in error tracking. A 02:00 import that never starts belongs in heartbeat monitoring.

## How should a NestJS error tracking filter capture HTTP exceptions?

Use an interceptor to establish request correlation and a global exception filter to report once and form the response. Don't report the same exception in both. One defect becoming two issue groups makes rollback evidence noisier precisely when time is scarce.

The adapter below is intentionally small. It uses the verified capture route, sends the API key from the environment, sets an explicit method, and retries HTTP 429 responses at most three times. The idempotency key remains stable across those retries.

```ts
import {
  ArgumentsHost,
  CallHandler,
  Catch,
  ExceptionFilter,
  ExecutionContext,
  HttpException,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { randomUUID } from "node:crypto";
import { Observable } from "rxjs";

type Boundary = "http" | "cron" | "worker" | "process";

type IncidentEvidence = {
  boundary: Boundary;
  operation: string;
  correlationId: string;
  error: Error;
  subjectId?: string;
  attempt?: number;
};

@Injectable()
export class IncidentReporter {
  async capture(event: IncidentEvidence): Promise<void> {
    const apiKey = process.env.INFRAI_API_KEY;
    const release = process.env.RELEASE_SHA;
    if (!apiKey || !release) {
      throw new Error("INFRAI_API_KEY and RELEASE_SHA are required");
    }

    const payload = {
      error_type: event.error.name,
      message: event.error.message,
      stack_trace: event.error.stack,
      environment: process.env.NODE_ENV ?? "production",
      release,
      context: {
        boundary: event.boundary,
        operation: event.operation,
        correlation_id: event.correlationId,
        subject_id: event.subjectId,
        attempt: event.attempt,
      },
    };
    const idempotencyKey = [release, event.boundary, event.correlationId].join(":");

    for (let retry = 0; retry < 3; retry += 1) {
      const apiOrigin = ["https://api", "infrai", "cc/v1"].join(".");
      const response = await fetch(`${apiOrigin}/errors/capture`, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
        },
        body: JSON.stringify(payload),
      });

      if (response.ok) return;

      const detail = await response.text();
      if (response.status !== 429 || retry === 2) {
        throw new Error(`Error capture failed (${response.status}): ${detail}`);
      }

      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** retry;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
  }
}

@Injectable()
export class CorrelationInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const request = context.switchToHttp().getRequest<{ id?: string }>();
    request.id ??= randomUUID();
    return next.handle();
  }
}

@Catch()
export class GlobalIncidentFilter implements ExceptionFilter {
  constructor(private readonly reporter: IncidentReporter) {}

  async catch(cause: unknown, host: ArgumentsHost): Promise<void> {
    const http = host.switchToHttp();
    const request = http.getRequest<{ id?: string; route?: { path?: string } }>();
    const response = http.getResponse<{
      status(code: number): { json(body: unknown): void };
    }>();
    const error = cause instanceof Error ? cause : new Error(String(cause));
    const status = cause instanceof HttpException ? cause.getStatus() : 500;

    await this.reporter.capture({
      boundary: "http",
      operation: request.route?.path ?? "unknown-route",
      correlationId: request.id ?? randomUUID(),
      error,
    });

    response.status(status).json({ statusCode: status });
  }
}
```

Requiring `RELEASE_SHA` is deliberate. Recording `unknown` would keep ingestion alive but weaken the exact decision this design serves. Fail deployment validation instead.

## Cron and worker failures need explicit ownership

Neither path crosses the HTTP filter. Wrap the top-level scheduled method and queue processor, attach the stable run or job ID, report the error, then rethrow it. Rethrowing preserves the scheduler's or queue's normal failure semantics.

```ts
import { Injectable } from "@nestjs/common";
import { Cron } from "@nestjs/schedule";

type TicketImportJob = {
  id: string;
  tenantId: string;
  attempt: number;
};

@Injectable()
export class TicketImportTasks {
  constructor(private readonly reporter: IncidentReporter) {}

  @Cron("0 */5 * * * *")
  async scheduleImports(): Promise<void> {
    const runId = `ticket-import:${new Date().toISOString().slice(0, 16)}`;
    try {
      await this.enqueueDueImports(runId);
    } catch (cause: unknown) {
      await this.reporter.capture({
        boundary: "cron",
        operation: "schedule-ticket-imports",
        correlationId: runId,
        error: cause instanceof Error ? cause : new Error(String(cause)),
      });
      throw cause;
    }
  }

  async process(job: TicketImportJob): Promise<void> {
    try {
      await this.importOnce(job.tenantId, job.id);
    } catch (cause: unknown) {
      await this.reporter.capture({
        boundary: "worker",
        operation: "import-support-tickets",
        correlationId: job.id,
        subjectId: job.tenantId,
        attempt: job.attempt,
        error: cause instanceof Error ? cause : new Error(String(cause)),
      });
      throw cause;
    }
  }

  private async enqueueDueImports(_runId: string): Promise<void> {
    return Promise.resolve();
  }

  private async importOnce(_tenantId: string, _jobId: string): Promise<void> {
    return Promise.resolve();
  }
}
```

The six-field worker context is more useful than a huge payload: boundary, operation, release, correlation ID, tenant ID, and attempt. Five-minute scheduling adds a separate monitoring requirement. A heartbeat service can expect a success signal for each interval and apply a grace window suited to normal queue delay; error tracking alone cannot observe a missing invocation.

Register `unhandledRejection` and `uncaughtException` handlers as the final boundary. After an uncaught exception, make a bounded attempt to flush evidence and terminate so the process supervisor can restart it. Continuing to serve support traffic from unknown process state is the risky option.

## When is the runner-up better?

Sentry should move to the front when source-map processing, release-centered issue triage, distributed tracing, or Session Replay controls the decision. Datadog is the stronger fit when errors must join an established infrastructure and APM operation. Better Stack deserves evaluation when uptime checks and incident response need to sit close to logs. Grafana works well for a team already prepared to operate the telemetry path and define its own dashboards.

Infrai occupies a narrower position. Its one REST boundary and public, self-describing discovery reduce integration upkeep, and consolidating credentials and billing removes recurring administration. But it has no native threshold rules, phone, SMS, or webhook notification routing. Alerts require polling recent unresolved groups and delivering notifications yourself. It also lacks heartbeat or synthetic monitoring, source-map decoding, crash symbolication, Session Replay, and distributed trace queries with span trees; trace and span IDs can only correlate log records.

These are material limitations and trade-offs, not footnotes. I would accept custom polling only when its maintenance fits inside the weekly shipping rhythm. If notification routing and rich triage save more operator time than a consolidated backend surface, pick Sentry, Datadog, or Better Stack after testing the workflow. If the team already owns collection and dashboards, Grafana may be the cleaner runner-up. The revenue-per-hour test is blunt: outsource the undifferentiated capability that consumes the most weekly attention.

Ship weekly, but keep the adapter yours. A small evidence interface makes a backend change contained, preserves the customer-support vocabulary, and leaves the rollback decision independent of a dashboard's data model.

## Further reading

- [NestJS exception filters](https://docs.nestjs.com/exception-filters)
- [NestJS task scheduling](https://docs.nestjs.com/techniques/task-scheduling)
- [Node.js process events](https://nodejs.org/api/process.html#process_event_uncaughtexception)
- [Sentry JavaScript source maps](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Datadog Error Tracking](https://docs.datadoghq.com/error_tracking/)
- [Better Stack uptime monitoring](https://betterstack.com/docs/uptime/)
- [Grafana error tracking](https://grafana.com/docs/grafana-cloud/monitor-applications/frontend-observability/instrument/error-tracking/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
