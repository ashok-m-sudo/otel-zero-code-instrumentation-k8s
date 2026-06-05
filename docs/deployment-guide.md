# Deployment guide

Follow these steps in order. Assume the sample workload from
[node-microservice-template](https://github.com/ashok-m-sudo/node-microservice-template)
is already deployed to the `microservices` namespace
(Deployments: `api-gateway`, `auth-service`, `backend-service`).

All commands are run from the root of this repository. Once everything is up,
head to [verification.md](verification.md) to prove a trace spans all three
services.

## 1. Create the observability namespace

All tracing infrastructure (collector, Jaeger, the Instrumentation CR) lives in a dedicated `observability` namespace.

```bash
kubectl apply -f manifests/00-namespace.yaml
```

## 2. Install cert-manager

The OTel Operator's mutating admission webhook is served over TLS. cert-manager issues and rotates those certificates; without it the operator never becomes Ready and no injection happens.

```bash
bash manifests/01-install-cert-manager.sh
```

Verify all cert-manager pods are Running:

```bash
kubectl get pods -n cert-manager
```

## 3. Install the OpenTelemetry Operator

The operator is the control-plane component that watches Deployments, injects the auto-instrumentation init container, and reconciles the Collector and Instrumentation custom resources.

```bash
bash manifests/02-install-otel-operator.sh
```

Verify the operator is Running:

```bash
kubectl get pods -n opentelemetry-operator-system
```

> The script waits on `deployment/opentelemetry-operator-controller-manager` in the `opentelemetry-operator-system` namespace. If a future operator release changes those names, adjust the `kubectl wait` line in the script accordingly.

## 4. Deploy Jaeger all-in-one

Jaeger receives spans over OTLP (port 4318) and serves the trace-search UI (port 16686). The manifest creates the Deployment plus two Services: `jaeger-collector` (ingest) and `jaeger-query` (UI).

```bash
kubectl apply -f manifests/04-jaeger-all-in-one.yaml
```

Verify the pod is Running and the services have a ClusterIP:

```bash
kubectl get pods,svc -n observability -l app=jaeger
```

## 5. Deploy the OTel Collector

The `OpenTelemetryCollector` custom resource is reconciled **by the operator** into a Deployment + Service. Because the CR is named `central-otel`, the generated service is `central-otel-collector` — which is exactly the OTLP endpoint the Instrumentation CR exports to.

```bash
kubectl apply -f manifests/03-otel-collector.yaml
```

Verify the operator created the collector workload:

```bash
kubectl get opentelemetrycollector,deploy,svc -n observability
# Expect a deployment and service named central-otel-collector
```

## 6. Create the Instrumentation resource

This declares *how* Node.js workloads should be instrumented: where to export spans, which propagators to use, and the sampling rate. It is the resource the inject annotation points at.

```bash
kubectl apply -f manifests/05-instrumentation.yaml
```

Verify it exists:

```bash
kubectl get instrumentation -n observability
```

## 7. Annotate the workload Deployments

This is the zero-code step. Patching each Deployment's pod template with the `instrumentation.opentelemetry.io/inject-nodejs` annotation tells the operator to inject the auto-instrumentation init container into every new pod. The patch triggers a rolling restart, so existing pods are replaced with instrumented ones automatically.

```bash
bash manifests/06-annotate-workloads.sh
```

The script patches all three Deployments (`api-gateway`, `auth-service`, `backend-service`) in the `microservices` namespace with the annotation value `observability/node-instrumentation`.

Watch the rollout complete:

```bash
kubectl rollout status deployment/api-gateway -n microservices
kubectl rollout status deployment/auth-service -n microservices
kubectl rollout status deployment/backend-service -n microservices
```

Once the new pods are Running, proceed to **[verification.md](verification.md)** to generate traffic and confirm a trace spans all three services.
