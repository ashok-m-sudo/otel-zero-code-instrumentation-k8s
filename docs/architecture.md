# Architecture

This document is a deep dive into the components that make zero-code tracing work and how telemetry flows between them. For step-by-step install instructions see [deployment-guide.md](deployment-guide.md); for proving it works see [verification.md](verification.md).

## Component inventory

| Component             | Role                                                                                                          | Namespace                        |
|-----------------------|--------------------------------------------------------------------------------------------------------------|----------------------------------|
| cert-manager          | Issues and rotates the TLS certificates used by the OTel Operator's admission webhook.                        | `cert-manager`                   |
| OpenTelemetry Operator| Control-plane controller. Watches Deployments for the inject annotation and mutates pod specs; reconciles the Collector and Instrumentation CRs. | `opentelemetry-operator-system`  |
| Instrumentation CRD   | Declarative spec for *how* to instrument a language runtime — exporter endpoint, propagators, sampler.        | `observability`                  |
| OTel Collector        | Receives OTLP spans from the workload, filters probe/metrics noise, batches, and forwards to Jaeger.          | `observability`                  |
| Jaeger all-in-one     | Receives spans (OTLP), stores them in memory, and serves the trace-search UI.                                 | `observability`                  |

## The auto-injection mechanism

The OpenTelemetry Operator registers a **mutating admission webhook** with the Kubernetes API server. Every time a pod is about to be created, the API server calls this webhook, giving the operator a chance to rewrite the pod spec before it is persisted.

The operator looks for the annotation `instrumentation.opentelemetry.io/inject-nodejs` on the pod template. When it finds one — for example on the `api-gateway` Deployment — it does three things to the pod spec, **without ever touching application source code or the container image**:

1. **Adds an init container** named `opentelemetry-auto-instrumentation`. This init container's only job is to copy the prebuilt Node.js OpenTelemetry SDK (auto-instrumentations bundle) into a shared `emptyDir` volume.
2. **Mounts that volume** into the application container so the SDK is visible on its filesystem at runtime.
3. **Injects environment variables** — most importantly `NODE_OPTIONS=--require /otel-auto-instrumentation-nodejs/autoinstrumentation.js`, which tells Node.js to load the OTel SDK before any application module runs. It also sets the OTLP exporter endpoint, the propagators, the sampler, and the service name — all sourced from the referenced `Instrumentation` resource.

Because the SDK is loaded via `--require` at process startup, it monkey-patches Node's `http`/`https` modules and popular libraries (Express, `axios`, etc.) automatically. Inbound requests create server spans; outbound requests create client spans and inject propagation headers. The application code is none the wiser.

The annotation value is `observability/node-instrumentation` — `<namespace>/<name>` of the `Instrumentation` resource. The namespace prefix matters: the pod being instrumented lives in `microservices`, but the `Instrumentation` CR lives in `observability`, so the operator must be told to look across namespaces.

## Context propagation

A trace only spans services if each service can hand its trace context to the next over the wire. That handoff is governed by **propagators** — the codecs that serialize/deserialize context into HTTP headers. We configure three, and every service must agree on the list:

```yaml
propagators:
  - tracecontext   # W3C Trace Context  — traceparent / tracestate headers
  - baggage        # W3C Baggage        — user-defined key/value pairs
  - b3             # Zipkin B3          — interop with B3-only services
```

- **`tracecontext`** is the [W3C standard](https://www.w3.org/TR/trace-context/). It carries the trace ID, parent span ID, and sampling flag in the `traceparent` header, plus vendor data in `tracestate`. This is the primary mechanism.
- **`baggage`** carries arbitrary user-defined key/value pairs ([W3C Baggage](https://www.w3.org/TR/baggage/)) in the `baggage` header — e.g. a `user.id` or `tenant` that you want visible on every downstream span. It does *not* affect trace structure; it rides alongside.
- **`b3`** is the older Zipkin propagation format. We include it so that if a service in the mesh only understands B3, context still flows. With multiple propagators configured, the SDK *injects all of them on send* and *accepts any of them on receive*, which maximizes interop at a small header-size cost.

Made concrete with the workload in this repo: when `api-gateway` proxies an inbound request to `auth-service`, its SDK injects `traceparent` (and `tracestate`, `baggage`, B3 headers) onto the outbound HTTP request. `auth-service`'s SDK reads `traceparent` from the incoming request, extracts the trace ID and the caller's span ID, and starts its server span as a **child** of that span — same trace ID, parent pointing at the gateway's span. The same handoff happens again when `backend-service` calls `auth-service`'s `/auth/verify` endpoint. Stitch those together and Jaeger shows one trace across all three services.

## Sampling strategy

The Instrumentation CR uses:

```yaml
sampler:
  type: parentbased_traceidratio
  argument: "1.0"
```

`parentbased_traceidratio` means: **if there is an upstream sampling decision, honor it; otherwise, decide based on the trace ID ratio.** The `argument` is that ratio. At `1.0` every root trace is sampled — 100% — which is what you want for a demo where you need to see every request you generate.

The "parent-based" part is important for consistency: once `api-gateway` decides a trace is sampled, that decision is encoded in the `traceparent` sampling flag and every downstream service honors it. You never get a half-sampled trace where the gateway span exists but the `auth-service` span was independently dropped.

In production you would lower the ratio (e.g. `"0.1"` for 10%) to control cost and storage, or move sampling decisions to the Collector with tail-based sampling so you can keep all error/slow traces while sampling the rest — that more advanced setup is out of scope here and belongs to the sibling repos.

## Why Jaeger all-in-one

This repo intentionally uses the `jaegertracing/jaeger:latest` **all-in-one** image: a single pod with **in-memory storage**. That keeps the moving parts to a minimum so the story stays on the instrumentation, not the trace store. The trade-off is that traces are lost when the pod restarts and there is no horizontal scaling — perfectly fine for a demo, unacceptable for production.

A production deployment would run Jaeger v2 with a durable backend (e.g. Elasticsearch or Cassandra), separate collector/query components, and retention policies. That production-grade trace store is the subject of the planned **ceph-distributed-tracing-jaeger-v2** sibling repo.
