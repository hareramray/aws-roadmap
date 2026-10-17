# Trace one request across a worker boundary

[Domain index](README.md) · [All tutorials](../README.md)

You will create local OpenTelemetry spans, propagate context through a simulated message, and demonstrate how instrumentation can mislead when context is lost. Then you will map that evidence to AWS tracing and Prometheus/Grafana choices. The baseline sends no telemetry to AWS and incurs no cloud charges.

## 1. Prepare isolated instrumentation

Create a Python virtual environment. Install compatible `opentelemetry-api` and `opentelemetry-sdk` packages from their official distribution, record the resolved versions in a lock or requirements file, and keep the experiment separate from a production application. The sample uses a console exporter so the trace leaves only local stdout.

Create `trace_order.py`:

```python
import json
import time
from opentelemetry import trace
from opentelemetry.context import Context
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor, ConsoleSpanExporter
from opentelemetry.trace import Status, StatusCode
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator

provider = TracerProvider(resource=Resource.create({
    "service.name": "tutorial-orders",
    "deployment.environment.name": "local",
}))
provider.add_span_processor(SimpleSpanProcessor(ConsoleSpanExporter()))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("tutorial")
propagator = TraceContextTextMapPropagator()

def worker(message, headers, fail=False):
    parent = propagator.extract(headers, context=Context())
    with tracer.start_as_current_span("reserve_inventory", context=parent) as span:
        span.set_attribute("order.id", message["order_id"])
        time.sleep(0.01)
        if fail:
            error = RuntimeError("synthetic inventory timeout")
            span.record_exception(error)
            span.set_status(Status(StatusCode.ERROR, str(error)))
            return "pending_retry"
        return "reserved"

def checkout(order_id, fail=False, lose_context=False):
    with tracer.start_as_current_span("checkout") as span:
        span.set_attribute("http.route", "/orders")
        headers = {}
        propagator.inject(headers)
        message = {"order_id": order_id}
        outcome = worker(message, {} if lose_context else headers, fail)
        ctx = span.get_span_context()
        print(json.dumps({"order_id": order_id, "outcome": outcome,
                          "trace_id": f"{ctx.trace_id:032x}"}))
        return headers

checkout("order-ok")
checkout("order-timeout", fail=True)
checkout("order-no-context", lose_context=True)
provider.shutdown()
```

Run it and save stdout. Expect three checkout spans and three worker spans. For the first two orders, the child span should share its parent's trace ID. The third worker receives an explicitly empty extracted context and becomes a separate trace. The missing relationship is an instrumentation defect even though the worker may complete successfully.

## 2. Read the trace as evidence

Inspect `name`, `context.trace_id`, `parent_id`, resource attributes, timestamps, status, and events. The synthetic failure is represented by an error status and exception event on `reserve_inventory`. The checkout span itself does not automatically become an error: decide whether returning “pending retry” violates the API's contract before assigning its status.

Write a record for each order containing expected business outcome, actual outcome, trace ID, and the span that explains the delay. Do not infer customer failure solely from every exception event; a handled retry may still produce a correct timely response. Conversely, a successful HTTP span can conceal an incorrect business record.

Replace the empty headers in the third call with the injected carrier and rerun. Verification means the worker shares the correct trace ID and has the expected parent. Merely seeing six spans in a backend does not prove context propagation works.

For real asynchronous work, put supported trace context into message metadata and validate its format. Understand whether the instrumentation creates a parent-child relationship or a span link for the messaging pattern. Batches can contain several originating traces; one arbitrary parent may misrepresent the work.

## 3. Make telemetry safe to export

The example's order IDs are synthetic. In a real system, choose attributes deliberately, redact confidential inputs, and normalize routes. Add deployment and environment identifiers at resource level where appropriate. Do not emit full SQL statements with secrets, authorization headers, customer addresses, or entire queue bodies.

Replace the simple exporter with a supported asynchronous/batch export path before production. Decide queue capacity, batch size, timeout, retry, and overflow behavior. The application must continue serving when telemetry cannot be exported. Test this by pointing a local exporter at an unavailable collector and confirming bounded memory, bounded logging, and acceptable request latency.

Learn sampling explicitly. Head sampling decides early and is inexpensive, but can miss a rare failure. Tail sampling can inspect completed traces but requires collection/state and correct routing. Calculate a volume estimate: 100 requests/second × 12 spans/request × 3,600 seconds gives 4.32 million spans/hour before sampling. A 10% sampling rate reduces the average exported count to about 432,000, but does not guarantee one tenth of every unusual event class.

Use metrics for aggregate rates and traces for diagnosis. Do not calculate a precise all-request failure percentage from a biased sample of error traces.

## 4. Route traces to AWS deliberately

For an optional extension, choose the current AWS-supported OpenTelemetry/CloudWatch collection path for the runtime. Draw application SDK → collector/agent → AWS destination, with its IAM role, network endpoints, TLS, buffering, and failure behavior. Use one small sandbox service and a short experiment window.

CloudWatch Application Signals can provide supported application metrics, maps, and SLO views. X-Ray remains relevant as a tracing backend, while the X-Ray SDKs/daemon entered maintenance in February 2026. Do not replace this local OpenTelemetry exercise with a new dependency on a legacy SDK merely because an older workshop uses it.

Check the current AWS Distro for OpenTelemetry instructions for your platform, including EKS/ECS/EC2 differences. A collector role needs only its intended export permissions. A collector also needs network reachability; granting IAM access does not create a route to an endpoint.

## 5. Add Prometheus-style metrics without exploding labels

Use these illustrative metrics to reason about aggregation:

```text
checkout_attempts_total{environment="tutorial",outcome="accepted"} 960
checkout_attempts_total{environment="tutorial",outcome="failed"} 20
checkout_attempts_total{environment="tutorial",outcome="invalid"} 20
```

These are cumulative counter examples, not a complete HTTP exporter. A live exporter must expose a valid endpoint, support the desired scrape format, and maintain counters consistently. Do not use request ID or order ID as labels. Three outcomes and two environments produce six label combinations; a million order IDs multiply that into a dangerous number of series.

For a live counter, the failure fraction query is conceptually:

```promql
sum(rate(checkout_attempts_total{outcome="failed"}[5m]))
/
sum(rate(checkout_attempts_total{outcome=~"accepted|failed"}[5m]))
```

The denominator deliberately excludes invalid inputs. A single static sample cannot demonstrate `rate`; collect repeated samples during controlled traffic and check counter resets. Add histogram buckets when latency percentiles are required, preserving the label grouping needed for correct aggregation.

Amazon Managed Service for Prometheus stores/query metrics using supported Prometheus workflows. Amazon Managed Grafana provides visualization and data-source integration. Compare them with CloudWatch according to existing PromQL skills, Kubernetes collection, cross-source dashboards, access control, and cost. Avoid duplicating every metric into several backends without a useful consumer.

## 6. Verify, troubleshoot, and clean up

Acceptance evidence includes a correctly connected successful trace, an error span, a repaired propagation defect, and a cardinality calculation. Explain what changes if the collector is unavailable, sampling is reduced, or an application instance restarts.

If spans do not appear, check SDK initialization order, active context, exporter configuration, resource naming, sampling, IAM, network paths, and backend time filters. If spans exist without metrics, do not assume one pipeline automatically enabled the other. If dashboards are empty, inspect scrape/export health before assuming the application is idle.

A CPU profile is a separate diagnostic tool: use it when a trace identifies expensive local code, rather than expecting spans to show individual function CPU allocation. Measure profiling overhead and protect profile artifacts.

For the local baseline, stop processes and keep only redacted outputs. For an AWS extension, remove collectors, temporary policies, log groups, Application Signals configuration where appropriate, dashboards, scrape jobs, and dedicated AMP/AMG workspaces. Review retention and ongoing ingestion; closing a dashboard does not stop telemetry charges.

References: [AWS Distro for OpenTelemetry](https://aws-otel.github.io/), [OpenTelemetry Python](https://opentelemetry.io/docs/languages/python/), [X-Ray lifecycle guidance](https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon.html), [Amazon Managed Service for Prometheus](https://docs.aws.amazon.com/prometheus/), [Amazon Managed Grafana](https://docs.aws.amazon.com/grafana/).
