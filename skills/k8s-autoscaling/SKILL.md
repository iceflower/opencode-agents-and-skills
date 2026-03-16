
# Kubernetes Autoscaling Guide

## 1. Autoscaling Tool Comparison

### Selection Criteria

| Feature | HPA | VPA | KEDA | Knative |
| --- | --- | --- | --- | --- |
| Scaling direction | Horizontal (replicas) | Vertical (resource limits) | Horizontal (replicas) | Horizontal (replicas) |
| Scale to zero | No | No | Yes | Yes |
| Event-driven | No | No | Yes (60+ scalers) | Yes (Eventing) |
| Traffic splitting | No | No | No | Yes |
| Metrics source | CPU, memory, custom | CPU, memory | External events, queues, DB, etc. | Concurrent requests |
| Revision management | No | No | No | Yes |
| Best for | Standard workloads | Right-sizing pods | Queue/event consumers | Serverless HTTP services |
| Complexity | Low | Low | Medium | High |

### When to Use What

| Scenario | Recommended |
| --- | --- |
| CPU/memory-based scaling | HPA |
| Pod resource optimization | VPA |
| Queue depth, event-driven batch | KEDA |
| Serverless HTTP with scale-to-zero | Knative Serving |
| Event routing and workflows | Knative Eventing |
| Scheduled scaling | KEDA (Cron scaler) |
| Canary/blue-green deployments | Knative (traffic splitting) |

---

## 2. KEDA

### 2.1. Overview

KEDA (Kubernetes Event-driven Autoscaling) extends Kubernetes HPA with event-driven scaling capabilities. It enables scaling based on external metrics like queue depth, database queries, and custom metrics.

```text
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   Event     │────>│    KEDA      │────>│     HPA     │
│   Source    │     │ Metrics API  │     │             │
└─────────────┘     └──────────────┘     └─────────────┘
                           │
                           v
                    ┌──────────────┐
                    │ ScaledObject │
                    └──────────────┘
```

### 2.2. Version Support (2026-03-14)

| Version | Release Date | Kubernetes Compatibility |
| --- | --- | --- |
| 2.20.x | Mar 2026 | 1.28+ |
| 2.19.x | Jan 2026 | 1.27+ |
| 2.18.x | Oct 2025 | 1.26+ |

### 2.3. Installation

```bash
# Add Helm repo
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

# Install KEDA
helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace

# Verify installation
kubectl get pods -n keda
kubectl get crd | grep keda
kubectl get apiservices | grep keda
```

### 2.4. ScaledObject

#### Basic Structure

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-app-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicaCount: 0   # Scale to zero
  maxReplicaCount: 20
  pollingInterval: 30  # Check every 30 seconds
  cooldownPeriod: 300  # Wait 5 minutes before scale down
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        topic: orders
        consumerGroup: my-app
        lagThreshold: "100"
```

#### ScaleTargetRef Fields

| Field | Description |
| --- | --- |
| `name` | Target resource name |
| `kind` | Deployment, StatefulSet, Custom Resource |
| `apiVersion` | Resource API version |
| `envSourceContainerName` | Container for env vars |

#### Replica Configuration

```yaml
spec:
  minReplicaCount: 0    # Minimum replicas (0 = scale to zero)
  maxReplicaCount: 20   # Maximum replicas
  idleReplicaCount: 1   # Replicas when no events (optional)
```

### 2.5. Common Scalers

#### Kafka

```yaml
triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      topic: orders
      consumerGroup: my-app
      lagThreshold: "100"
      offsetResetPolicy: latest
```

#### RabbitMQ

```yaml
triggers:
  - type: rabbitmq
    metadata:
      host: amqp://guest:guest@rabbitmq:5672/vhost
      queueName: orders
      mode: QueueLength
      value: "100"
```

#### AWS SQS

```yaml
triggers:
  - type: aws-sqs
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789/orders
      queueLength: "100"
      awsRegion: us-east-1
    authenticationRef:
      name: aws-credentials
```

#### Redis

```yaml
triggers:
  - type: redis
    metadata:
      address: redis:6379
      listName: orders
      listLength: "50"
```

#### PostgreSQL

```yaml
triggers:
  - type: postgresql
    metadata:
      host: postgres.default.svc.cluster.local
      port: "5432"
      userName: postgres
      dbName: orders
      passwordFromEnv: POSTGRES_PASSWORD
      query: "SELECT COUNT(*) FROM pending_orders"
      targetQueryValue: "100"
```

#### Prometheus

```yaml
triggers:
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: http_requests_total
      threshold: "100"
      query: "sum(rate(http_requests_total[1m]))"
```

#### Cron

```yaml
triggers:
  - type: cron
    metadata:
      timezone: Asia/Seoul
      start: "0 9 * * *"   # Scale up at 9 AM
      end: "0 18 * * *"     # Scale down at 6 PM
      desiredReplicas: "10"
```

#### CPU/Memory (Native HPA)

```yaml
triggers:
  - type: cpu
    metadata:
      type: Utilization
      value: "70"
  - type: memory
    metadata:
      type: Utilization
      value: "80"
```

#### Multi-Scaler

Combine multiple triggers:

```yaml
triggers:
  # Primary: Queue depth
  - type: rabbitmq
    metadata:
      host: amqp://rabbitmq:5672
      queueName: orders
      mode: QueueLength
      value: "100"
  # Secondary: CPU utilization
  - type: cpu
    metadata:
      type: Utilization
      value: "70"
```

### 2.6. Authentication

#### TriggerAuthentication

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: aws-credentials
spec:
  secretTargetRef:
    - parameter: awsAccessKeyID
      name: aws-secret
      key: AWS_ACCESS_KEY_ID
    - parameter: awsSecretAccessKey
      name: aws-secret
      key: AWS_SECRET_ACCESS_KEY
```

#### ClusterTriggerAuthentication

```yaml
apiVersion: keda.sh/v1alpha1
kind: ClusterTriggerAuthentication
metadata:
  name: aws-credentials
spec:
  podIdentity:
    provider: aws-eks  # Use IRSA
```

#### Authentication Providers

| Provider | Use Case |
| --- | --- |
| `secretTargetRef` | Kubernetes secrets |
| `env` | Environment variables |
| `podIdentity` | Cloud IAM (EKS, AKS, GKE) |
| `hashiCorpVault` | Vault secrets |
| `azureKeyVault` | Azure Key Vault |
| `awsSecretManager` | AWS Secrets Manager |

#### Example: IRSA on EKS

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: aws-auth
spec:
  podIdentity:
    provider: aws-eks
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: sqs-scaler
spec:
  scaleTargetRef:
    name: my-app
  triggers:
    - type: aws-sqs
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/xxx/orders
        awsRegion: us-east-1
      authenticationRef:
        name: aws-auth
```

### 2.7. ScaledJob

For batch processing workloads:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: order-processor
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: processor
            image: my-app:latest
            command: ["./process-orders"]
        restartPolicy: OnFailure
        backoffLimit: 3
  pollingInterval: 30
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: kafka:9092
        topic: orders
        consumerGroup: processor
        lagThreshold: "10"
```

#### ScaledJob vs ScaledObject

| Feature | ScaledObject | ScaledJob |
| --- | --- | --- |
| Target | Deployment, StatefulSet | Job |
| Use case | Long-running services | Batch processing |
| Scaling | Pod replicas | Job instances |
| Scale to zero | Yes | Yes |

### 2.8. Advanced Configuration

#### Scaling Behavior

```yaml
spec:
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300
          policies:
            - type: Percent
              value: 10
              periodSeconds: 60
        scaleUp:
          stabilizationWindowSeconds: 0
          policies:
            - type: Percent
              value: 100
              periodSeconds: 15
```

#### Custom Metrics

```yaml
triggers:
  - type: external
    metadata:
      scalerAddress: custom-scaler:6000
      metricName: custom_metric
      targetValue: "10"
```

### 2.9. KEDA Monitoring

| Metric | Description |
| --- | --- |
| `keda_scaledobject_ready` | ScaledObject ready status |
| `keda_scaledobject_errors` | Scaling errors |
| `keda_scaler_errors` | Scaler-specific errors |
| `keda_internal_scale_loop_latency` | Scaling latency |

Prometheus integration:

```bash
helm install keda kedacore/keda \
  --namespace keda \
  --set prometheus.metricServer.enabled=true \
  --set prometheus.operator.enabled=true
```

### 2.10. KEDA Troubleshooting

#### ScaledObject Not Scaling

```bash
# Check ScaledObject status
kubectl get scaledobject -o yaml

# Check HPA
kubectl get hpa

# Check KEDA metrics server
kubectl logs -n keda -l app=keda-operator-metrics-apiserver
```

#### Authentication Failures

```bash
# Check TriggerAuthentication
kubectl get triggerauthentication -o yaml

# Verify secrets exist
kubectl get secrets
```

#### Metrics Not Available

```bash
# Check KEDA operator logs
kubectl logs -n keda -l app=keda-operator

# Verify scaler connectivity
kubectl run test --rm -it --image=curlimages/curl -- curl http://scaler:port/metrics
```

#### Debug Commands

```bash
# List ScaledObjects
kubectl get scaledobject

# Describe ScaledObject
kubectl describe scaledobject my-scaler

# Check HPA created by KEDA
kubectl get hpa -l app.kubernetes.io/managed-by=keda-operator

# View events
kubectl get events --field-selector reason=KEDAScaleTarget
```

---

## 3. Knative

### 3.1. Overview

Knative brings serverless capabilities to Kubernetes:

- **Serving**: Auto-scaling deployments with scale-to-zero
- **Eventing**: Event-driven architecture framework

| Feature | Description |
| --- | --- |
| Scale-to-zero | No resources when idle |
| Auto-scaling | Based on concurrent requests |
| Traffic splitting | Blue/green, canary deployments |
| Event routing | Loose coupling with brokers/triggers |
| Revision history | Automatic versioning |

### 3.2. Version Support (2026-03-14)

| Version | Release Date | End of Support | Kubernetes |
| --- | --- | --- | --- |
| 1.21 | Jan 2026 | Aug 2026 | 1.33+ |
| 1.20 | Oct 2025 | Apr 2026 | 1.32+ |
| 1.19 | Jul 2025 | Jan 2026 | 1.32+ |

Release cadence: new minor version every 3 months, 6 months community support per release, only latest 2 releases supported.

### 3.3. Installation

#### Serving

```bash
# Install CRDs
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.21.0/serving-crds.yaml

# Install core components
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.21.0/serving-core.yaml

# Install networking layer (Kourier - recommended)
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.21.0/kourier.yaml

# Configure networking
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

#### Eventing

```bash
# Install CRDs
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.21.0/eventing-crds.yaml

# Install core components
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.21.0/eventing-core.yaml

# Install broker (MTChannelBasedBroker)
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.21.0/mt-channel-broker.yaml
```

#### Verify Installation

```bash
kubectl get pods -n knative-serving
kubectl get pods -n knative-eventing
```

### 3.4. Knative Serving

#### Service Definition

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: hello
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/min-scale: "0"
        autoscaling.knative.dev/max-scale: "10"
        autoscaling.knative.dev/target: "100"
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "World"
```

#### Autoscaling Annotations

| Annotation | Description | Default |
| --- | --- | --- |
| `autoscaling.knative.dev/min-scale` | Minimum replicas | 0 |
| `autoscaling.knative.dev/max-scale` | Maximum replicas | None |
| `autoscaling.knative.dev/target` | Target concurrency | 100 |
| `autoscaling.knative.dev/target-utilization-percentage` | Target utilization | 70 |
| `autoscaling.knative.dev/scale-down-delay` | Delay before scale down | 0s |
| `autoscaling.knative.dev/window` | Stable window | 60s |

#### Scale-to-Zero Configuration

```yaml
metadata:
  annotations:
    autoscaling.knative.dev/min-scale: "0"
    autoscaling.knative.dev/scale-to-zero-pod-retention-period: "5m"
```

#### Revision Management

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-app
spec:
  template:
    metadata:
      name: my-app-v2
    spec:
      containers:
        - image: my-app:v2
  traffic:
    - tag: current
      revisionName: my-app-v1
      percent: 90
    - tag: latest
      revisionName: my-app-v2
      percent: 10
```

### 3.5. Traffic Management

#### Traffic Splitting

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: my-app
spec:
  traffic:
    - tag: stable
      revisionName: my-app-v1
      percent: 90
    - tag: canary
      revisionName: my-app-v2
      percent: 10
      url: my-app-canary.default.example.com
```

#### Blue/Green Deployment

```yaml
spec:
  traffic:
    - tag: blue
      revisionName: my-app-blue
      percent: 100
    - tag: green
      revisionName: my-app-green
      percent: 0
```

#### Tag-Based Routing

```bash
# Access specific revision by tag
curl http://my-app-canary.default.example.com
```

### 3.6. Knative Eventing

#### Broker and Trigger

```yaml
# Create default broker
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: default
  namespace: production
---
# Trigger routes events to subscriber
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: order-processor
  namespace: production
spec:
  broker: default
  filter:
    attributes:
      type: order.created
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: order-processor
```

#### Event Sources

##### PingSource (Cron)

```yaml
apiVersion: sources.knative.dev/v1
kind: PingSource
metadata:
  name: hourly-cleanup
spec:
  schedule: "0 * * * *"
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: cleanup-job
```

##### ApiServerSource

```yaml
apiVersion: sources.knative.dev/v1
kind: ApiServerSource
metadata:
  name: pod-events
spec:
  mode: Resource
  resources:
    - apiVersion: v1
      kind: Pod
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: pod-monitor
```

##### KafkaSource

```yaml
apiVersion: sources.knative.dev/v1beta1
kind: KafkaSource
metadata:
  name: kafka-source
spec:
  consumerGroup: knative-group
  bootstrapServers:
    - kafka:9092
  topics:
    - orders
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: order-processor
```

### 3.7. Event Patterns

#### Channel-Based Messaging

```yaml
apiVersion: messaging.knative.dev/v1
kind: Channel
metadata:
  name: orders
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1
    kind: InMemoryChannel
---
apiVersion: messaging.knative.dev/v1
kind: Subscription
metadata:
  name: order-subscription
spec:
  channel:
    apiVersion: messaging.knative.dev/v1
    kind: Channel
    name: orders
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: order-processor
```

#### Sequence (Workflow)

```yaml
apiVersion: flows.knative.dev/v1
kind: Sequence
metadata:
  name: order-workflow
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1
    kind: InMemoryChannel
  steps:
    - ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: validate-order
    - ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: process-payment
    - ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: ship-order
```

### 3.8. Knative Configuration

#### Config-Autoscaler

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-autoscaler
  namespace: knative-serving
data:
  enable-scale-to-zero: "true"
  stable-window: "60s"
  scale-to-zero-grace-period: "30s"
  tick-interval: "2s"
  target-burst-capacity: "200"
```

#### Config-Defaults

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-defaults
  namespace: knative-serving
data:
  revision-timeout-seconds: "300"
  max-revision-timeout-seconds: "600"
  revision-cpu-request: "100m"
  revision-memory-request: "128Mi"
  revision-cpu-limit: "1000m"
  revision-memory-limit: "512Mi"
```

#### Config-Domain

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-domain
  namespace: knative-serving
data:
  example.com: ""
  myapp.example.com: |
    selector:
      app: myapp
```

### 3.9. Knative Security

#### Pod Security

```yaml
spec:
  template:
    spec:
      containers:
        - name: app
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
```

#### Secure Pod Defaults (v1.21+)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-features
  namespace: knative-serving
data:
  secure-pod-defaults: "AllowRootBounded"
```

#### Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: knative-serving
  namespace: knative-serving
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: knative-serving
```

### 3.10. Knative Monitoring

| Metric | Description |
| --- | --- |
| `knative_serving_request_count` | Total requests |
| `knative_serving_request_latencies` | Request latency |
| `knative_serving_revision_ready` | Revision ready status |
| `knative_eventing_broker_event_count` | Events through broker |

Grafana dashboard: `https://grafana.com/grafana/dashboards/13398`

### 3.11. Knative Troubleshooting

#### Service Not Ready

```bash
# Check service status
kubectl get ksvc -o yaml

# Check revision status
kubectl get revisions

# Check pods
kubectl get pods -l serving.knative.dev/service=hello
```

#### Scale-to-Zero Not Working

```bash
# Check autoscaler config
kubectl get configmap config-autoscaler -n knative-serving -o yaml

# Check activator
kubectl logs -n knative-serving -l app=activator
```

#### Event Not Delivered

```bash
# Check broker status
kubectl get broker -o yaml

# Check trigger status
kubectl get trigger -o yaml

# Check event delivery
kubectl logs -n knative-eventing -l app=broker-filter
```

#### Knative Debug Commands

```bash
# Describe service
kubectl describe ksvc hello

# Check revisions
kubectl get revisions -l serving.knative.dev/service=hello

# View events
kubectl get events --field-selector involvedObject.kind=Service

# Check ingress
kubectl get ingress -n knative-serving
```

---

## 4. Common Best Practices

### Scale-to-Zero Usage

- Good for: Batch processing, dev/staging environments, low-traffic services
- Avoid for: Production services with strict latency requirements (cold start penalty)
- Monitor cold start latency and set `idleReplicaCount: 1` (KEDA) or `min-scale: "1"` (Knative) to avoid cold starts when needed

### Resource Limits

Always set resource requests and limits on scaled workloads:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 1000m
    memory: 512Mi
```

### Scaling Thresholds

- Queue-based (KEDA): Start with 100-500 messages per replica
- CPU: 70-80% utilization
- Memory: 80% utilization
- Concurrency (Knative): Start with target 100

### Cooldown and Stabilization

- Configure cooldown periods to prevent flapping (KEDA: `cooldownPeriod`, Knative: `scale-down-delay`)
- Use stabilization windows for gradual scale-down

### Monitoring and Alerting

- Monitor scaling events across both KEDA and Knative

```bash
# KEDA scaling events
kubectl get events -A --field-selector reason=KEDAScaleTarget

# Knative cold start latency
kubectl logs -n knative-serving -l app=activator | grep "cold start"
```

- Set alerts on scaling errors and cold start latency
- Track queue depth / consumer lag trends to tune thresholds

### Testing

- Verify scaler connectivity before production deployment
- Test both scale-up and scale-down behavior
- Load test with realistic traffic patterns
- Validate behavior during event source failures

---

## 5. References

### KEDA

- [KEDA Documentation](https://keda.sh/docs/)
- [KEDA Scalers](https://keda.sh/docs/scalers/)
- [KEDA GitHub](https://github.com/kedacore/keda)

### Knative

- [Knative Documentation](https://knative.dev/docs/)
- [Serving API](https://knative.dev/docs/reference/api/serving-api/)
- [Eventing API](https://knative.dev/docs/reference/api/eventing-api/)
- [Autoscaling Config](https://knative.dev/docs/serving/configuring-autoscaling/)
