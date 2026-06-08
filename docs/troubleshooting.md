# Troubleshooting

Common failure modes when standing up zero-code tracing, with the underlying cause and the fix. Each entry follows **symptom → cause → fix**.

## 1. OTel Operator pod stuck in `Pending` / `CrashLoopBackOff`

**Symptom:** `kubectl get pods -n opentelemetry-operator-system` shows the controller manager never reaching `Running`/`Ready`; events mention failing to mount a certificate secret or the webhook failing TLS.

**Cause:** The operator's admission webhook needs TLS certificates that cert-manager provisions. If cert-manager isn't installed (or its `Certificate` hasn't been issued yet), the operator can't come up.

**Fix:** Install cert-manager first and confirm certs are issued.

```bash
bash manifests/01-install-cert-manager.sh
kubectl get pods -n cert-manager
kubectl get certificate -A
```

Then reinstall / restart the operator (`bash manifests/02-install-otel-operator.sh`).

## 2. Annotation applied but no init container injected

**Symptom:** `kubectl describe pod <pod> -n microservices` shows no `opentelemetry-auto-instrumentation` init container even though the Deployment carries the annotation.

**Cause:** Either the pods were never restarted after annotating (the webhook only fires on pod *creation*), **or** the annotation value omits the namespace prefix so the operator can't find the `Instrumentation` CR (which lives in `observability`, not in the workload namespace).

**Fix:** Ensure the annotation value is `<namespace>/<instrumentation-name>` — i.e. `observability/node-instrumentation` — and force a restart:

```bash
kubectl get deploy api-gateway -n microservices \
  -o jsonpath='{.spec.template.metadata.annotations}' ; echo
kubectl rollout restart deployment/api-gateway -n microservices
```

Repeat for `auth-service` and `backend-service`. Also confirm the operator pod is Running — if its webhook is down, no mutation happens.

## 3. Init container injects but no spans reach Jaeger

**Symptom:** Pods have the init container and start fine, but Jaeger shows no traces for the services.

**Cause:** The exporter endpoint in the `Instrumentation` CR (or the collector → Jaeger hop) is misconfigured — wrong service name, wrong port, or wrong protocol.

**Fix:** Confirm `spec.exporter.endpoint` points at the collector's OTLP/HTTP service by DNS and port:

```bash
kubectl get instrumentation node-instrumentation -n observability -o yaml | grep -A1 exporter
# endpoint: http://central-otel-collector.observability.svc.cluster.local:4318
```

Verify the collector service exists and listens on 4318, and that the collector is forwarding to Jaeger:

```bash
kubectl get svc central-otel-collector -n observability
kubectl logs -n observability deploy/central-otel-collector --tail=50
```

Confirm Jaeger is accepting OTLP on 4318:

```bash
kubectl get svc jaeger-collector -n observability
```

## 4. Spans appear but each service has its own disconnected trace

**Symptom:** Jaeger shows spans for each service, but they're in separate traces with different trace IDs — no waterfall across services.

**Cause:** Propagator mismatch. If the sender injects one propagation format and the receiver only reads another, the receiver can't extract the upstream context and starts a brand-new trace.

**Fix:** Ensure every service shares the **same** `propagators` list in the `Instrumentation` CR (this repo uses `tracecontext`, `baggage`, `b3` for all three), then restart the pods so they pick up the change:

```bash
kubectl get instrumentation node-instrumentation -n observability \
  -o jsonpath='{.spec.propagators}' ; echo
kubectl rollout restart deployment/api-gateway deployment/auth-service deployment/backend-service -n microservices
```

## 5. Traces appear but the service name is `unknown_service`

**Symptom:** Spans show up in Jaeger under `unknown_service` (or `unknown_service:node`) instead of `api-gateway` / `auth-service` / `backend-service`.

**Cause:** `OTEL_SERVICE_NAME` wasn't derived. The operator normally infers the service name from the pod's `app.kubernetes.io/name` label (or related labels); if your Deployments use a different label scheme, the inference fails.

**Fix:** The workload here labels pods with `app: <name>`. Either add the conventional label, or set the name explicitly. To make it deterministic, add an `env` block to the `Instrumentation` CR that derives the name from the Deployment via the downward API — for example:

```yaml
spec:
  env:
    - name: OTEL_SERVICE_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.labels['app']
```

Restart the pods afterward.

## 6. High pod startup latency after enabling injection

**Symptom:** Annotated pods take noticeably longer to become Ready than before.

**Cause:** The auto-instrumentation init container has to copy the Node.js OTel SDK into the shared volume at pod start (and, depending on image, fetch it). That work is paid on every pod start.

**Fix:** For latency-sensitive workloads, pre-bake the OTel SDK into a custom workload image instead of relying on the init-container copy at runtime, or pin the auto-instrumentation image to a locally cached tag. See the [OpenTelemetry Operator docs](https://github.com/open-telemetry/opentelemetry-operator#opentelemetry-auto-instrumentation-injection) for the supported image-override fields on the `Instrumentation` CR.
