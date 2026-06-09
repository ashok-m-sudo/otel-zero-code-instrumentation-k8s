# Verification

This document proves the setup works end to end: an init container is injected with no code changes, traffic flows through all three services, and Jaeger shows a single trace spanning `api-gateway`, `backend-service`, and `auth-service` with one shared trace ID.

> **The hero request.** `GET /api/backend/data` is the request that exercises all three services in one trace. The gateway proxies it to `backend-service`; `backend-service`'s auth middleware then calls `auth-service`'s `/auth/verify` to validate the token. One inbound request, two downstream hops, three services.

## 1. Confirm the init container was injected

Pick any pod from an annotated Deployment and describe it. With zero changes to the image or source, the operator should have added an init container.

```bash
POD=$(kubectl get pod -n microservices -l app=backend-service -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod "$POD" -n microservices
```

Look for an init container named **`opentelemetry-auto-instrumentation`** in the `Init Containers:` section, and OTel environment variables (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, `OTEL_PROPAGATORS`, `NODE_OPTIONS`) on the app container. A quick check:

```bash
kubectl get pod "$POD" -n microservices \
  -o jsonpath='{.spec.initContainers[*].name}{"\n"}'
# -> opentelemetry-auto-instrumentation
```

## 2. Generate sample traffic

First, register and log in to get a JWT, then call the backend with it. Port-forward the gateway in one terminal:

```bash
kubectl port-forward svc/api-gateway 8080:3000 -n microservices
```

In a second terminal, drive the hero flow a few times:

```bash
# Register a demo user (idempotent-ish; 409 on repeat is fine)
curl -s -X POST http://localhost:8080/api/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","password":"demo-pass","email":"demo@example.com"}'

# Log in and capture the token
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","password":"demo-pass"}' | sed -n 's/.*"token":"\([^"]*\)".*/\1/p')

# Hit the backend WITH the token a few times -> api-gateway -> backend-service -> auth-service
for i in 1 2 3 4 5; do
  curl -s http://localhost:8080/api/backend/data \
    -H "Authorization: Bearer $TOKEN" > /dev/null
  echo "request $i sent"
done
```

> If you only want a two-service trace (`api-gateway → auth-service`), `curl http://localhost:8080/api/auth/verify -H "Authorization: Bearer $TOKEN"` works too — but the backend call above is the one that produces the full three-service trace.

## 3. Access the Jaeger UI

Port-forward the Jaeger query service:

```bash
kubectl port-forward svc/jaeger-query 16686:16686 -n observability
```

Then open **http://localhost:16686** in your browser.

## 4. Find your trace

1. In the **Service** dropdown, select **`api-gateway`**.
2. Click **Find Traces**.
3. Open the most recent trace (the ones from your `/api/backend/data` calls).

You should see a waterfall with spans from all three services nested under the gateway's root span.

## 5. Acceptance criteria

- [ ] A single trace contains spans from all three services (`api-gateway`, `backend-service`, `auth-service`).
- [ ] All spans in that trace share the same `trace_id`.
- [ ] Each downstream span's `parent_span_id` correctly references the upstream span's `span_id` (gateway → backend, backend → auth).
- [ ] Span attributes include `http.method`, `http.status_code`, and `http.url` (exact attribute keys may vary slightly by SDK version, e.g. `http.request.method` / `url.full` under newer semantic conventions).

![Jaeger trace across three services](../images/jaeger-trace-three-services.png)

> **Screenshot TODO:** drop a capture of the three-service trace waterfall into `images/jaeger-trace-three-services.png` so this renders.

## 6. Inspect propagation headers (optional)

To *prove* the W3C `traceparent` header is being forwarded rather than each service starting a fresh trace, add a temporary debug log on a receiving service that prints the incoming header. For example, in `auth-service`'s request-logging middleware:

```js
app.use((req, res, next) => {
    logger.info(`traceparent: ${req.headers['traceparent'] || '(none)'}`);
    next();
});
```

Redeploy, generate traffic, then tail the logs:

```bash
kubectl logs -n microservices -l app=auth-service --tail=50 | grep traceparent
```

A line like `traceparent: 00-<32-hex-trace-id>-<16-hex-span-id>-01` confirms the upstream service injected W3C context and `auth-service` received it. The trace-id segment will match the `trace_id` you see in Jaeger. Remember to remove the debug log afterward — it is only a teaching aid, not part of the zero-code story.
