# OTel Collector + LocalStack Deployment POC

## Prerequisites

- `kubectl`
- `minikube` for setting up a local environment ([installation guide](https://minikube.sigs.k8s.io/docs/start/?arch=%2Fmacos%2Farm64%2Fstable%2Fbinary+download))
- `otel-cli` for generating/sending test traces ([installation guide](https://github.com/equinix-labs/otel-cli#installation))
- `aws-cli` for creating an s3 bucket in localstack

---

## Deployment

1. Apply the Kubernetes Deployment:

```bash
kubectl apply -f poc-deployment.yaml
```

This deploys:
- LocalStack (S3 service)
- OpenTelemetry Collector (with AWS S3 exporter)
- A ConfigMap for the collector’s configuration

Verify deployments:

```bash
kubectl get pods -n observability
```

Expected output:
```
NAME                             READY   STATUS    RESTARTS   AGE
localstack-xxxxxxxxx-xxxxx       1/1     Running   0          1m
otel-collector-xxxxxxxxx-xxxxx   1/1     Running   0          1m
```

---

## Port-Forward LocalStack

This step allows us to access the localstack pod, and create a bucket using `aws-cli`.

1. Port-forward LocalStack:

```bash
kubectl port-forward -n observability svc/localstack 4566:4566
```

Keep this terminal open.

2. Verify LocalStack is accessible, the following command displays all the available s3 buckets:

In a new terminal:

```bash
aws --endpoint-url=http://localhost:4566 s3 ls
```

This should return nothing as we haven't created any buckets yet.

---

## Create an S3 Bucket

1. Create the `otel-traces` S3 Bucket:

```bash
aws --endpoint-url=http://localhost:4566 --region eu-west-1 s3 mb s3://otel-traces
```

2. Verify the Bucket was created:

```bash
aws --endpoint-url=http://localhost:4566 s3 ls
```

Expected output:
```
2024-02-06 12:00:00 otel-traces
```

---

## Port-Forward OpenTelemetry Collector

This step allows us to access the OTel Collector pod, and generate test traces.

1. Port-forward the OTLP Endpoint:

```bash
kubectl port-forward -n observability svc/otel-collector 4317:4317
```

Keep this terminal open.

---

## Send a Test Trace Using `otel-cli`

1. Send a Simple Test Trace:

In a new terminal:

```bash
otel-cli span --name "test-span" --endpoint http://localhost:4317 --protocol grpc --verbose
```

2. Verify the Trace in S3:

```bash
aws --endpoint-url=http://localhost:4566 s3 ls s3://otel-traces --recursive
```

Expected output:
```
2024-02-06 12:15:00   otel-collector/traces/year=2025/month=02/day=12/hour=10/minute=36/traces_123.json
```

3. To view the Trace file:

```bash
aws --endpoint-url=http://localhost:4566 s3 cp s3://otel-traces/otel-collector/traces/year=2025/month=02/day=12/hour=10/minute=36/traces_123.json -
```

---

## Troubleshooting

1. Check LocalStack Logs:

```bash
kubectl logs -n observability -l app=localstack
```

2. Check OpenTelemetry Collector Logs:

```bash
kubectl logs -n observability -l app=otel-collector
```

3. Verify Services:

```bash
kubectl get svc -n observability
```

4. Check Pod Status:

```bash
kubectl get pods -n observability
```

add test1 to readme