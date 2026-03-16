---
name: karpenter-workflow
description: >-
  Karpenter core workflow, NodePool/EC2NodeClass configuration,
  autoscaling patterns, and troubleshooting. Use for general Karpenter operations.
---

# Karpenter Workflow Guide

## 1. Overview

Karpenter is an open-source Kubernetes node autoscaler designed for flexibility, performance, and simplicity. Unlike Cluster Autoscaler, Karpenter provisions nodes based on pod requirements rather than node group configurations.

### Key Features

- **Just-in-time provisioning**: Nodes created when pods are pending
- **Flexible scheduling**: Considers CPU, memory, GPU, storage, and topology
- **Consolidation**: Automatically replaces nodes with cheaper alternatives
- **Multi-cloud support**: AWS, Azure, and GKE

### Karpenter vs Cluster Autoscaler

| Feature | Karpenter | Cluster Autoscaler |
| --- | --- | --- |
| Scaling model | Pod-driven | Node group-driven |
| Instance selection | Dynamic | Pre-configured groups |
| Spot support | Native | Limited |
| Consolidation | Active | Passive |
| Cloud support | AWS, Azure, GKE | All major clouds |

---

## 2. Version Support (2026-03-14)

### Latest Version

| Version | Release Date | Kubernetes Compatibility |
| --- | --- | --- |
| v1.0.x | Feb 2026 | 1.31+ |
| v0.37.x | Oct 2025 | 1.30 |
| v0.34.x | Jul 2025 | 1.29 |
| v0.31.x | Apr 2025 | 1.28 |

### Compatibility Matrix

| Kubernetes | Karpenter |
| --- | --- |
| 1.31 | v1.0.5+ |
| 1.30 | v0.37.x |
| 1.29 | v0.34.x |
| 1.28 | v0.31.x |
| 1.27 | v0.28.x |
| 1.25 | v0.25.x |

---

## 3. Core Concepts

### NodePool

Defines where and how Karpenter should provision nodes.

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```

### EC2NodeClass / AKSNodeClass / GKENodeClass

Provider-specific node configuration.

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiSelectorTerms:
    - alias: al2023@latest
  role: KarpenterNodeRole
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster
  tags:
    Environment: production
```

---

## 4. Autoscaling Configuration

### Requirements

Define node constraints using Kubernetes label selectors:

| Requirement Key | Values | Description |
| --- | --- | --- |
| `karpenter.sh/capacity-type` | spot, on-demand | Instance market type |
| `kubernetes.io/arch` | amd64, arm64 | CPU architecture |
| `kubernetes.io/os` | linux, windows | Operating system |
| `karpenter.k8s.aws/instance-category` | c, m, r, etc. | Instance family |
| `karpenter.k8s.aws/instance-generation` | 5, 6, 7 | Instance generation |

```yaml
spec:
  template:
    spec:
      requirements:
        # Capacity type
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]

        # Instance types
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m6i.xlarge", "m6i.2xlarge", "m7i.xlarge"]

        # Architecture
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]

        # Exclude small instances
        - key: karpenter.k8s.aws/instance-size
          operator: NotIn
          values: ["nano", "micro", "small"]
```

### Resource Limits

```yaml
spec:
  limits:
    cpu: 1000      # Max 1000 vCPUs
    memory: 1000Gi # Max 1000GiB memory
```

---

## 5. Disruption Policies

### Consolidation Policies

| Policy | Description |
| --- | --- |
| `WhenEmpty` | Replace empty nodes only |
| `WhenEmptyOrUnderutilized` | Replace underutilized nodes (recommended) |

```yaml
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
    budgets:
      - nodes: 10%
        schedule: "0 0 * * *"  # Allow 10% disruption at midnight
      - nodes: 0               # No disruption during business hours
        schedule: "0 9-17 * * MON-FRI"
```

### Expiration

```yaml
spec:
  disruption:
    expireAfter: 720h  # Nodes expire after 30 days
```

---

## 6. Pod Scheduling

### Node Affinity

Karpenter considers pod scheduling constraints:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-east-1a", "us-east-1b"]
```

### Topology Spread

```yaml
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app: myapp
```

### Priority Classes

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: high-priority
spec:
  template:
    spec:
      # Higher priority pods get dedicated nodes
```

---

## 7. Spot Instance Handling

### Spot Interruption

Karpenter automatically handles spot interruptions:

1. **2-minute warning**: AWS sends spot interruption notice
2. **Cordon and drain**: Karpenter cordons node and drains pods
3. **Replacement**: New node provisioned before termination

### Spot Diversification

```yaml
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m6i", "m7i", "c6i", "c7i"]  # Multiple families
```

### Spot Fallback

```yaml
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]  # Fall back to on-demand
```

---

## 8. Node Initialization

### Init Containers

```yaml
spec:
  template:
    spec:
      startupTaints:
        - key: node.kubernetes.io/not-ready
          effect: NoSchedule
      initContainers:
        - name: init
          image: busybox
          command: ["/bin/sh", "-c", "echo initializing"]
```

### UserData

```yaml
spec:
  userData: |
    #!/bin/bash
    echo "Custom node initialization"
    /etc/eks/bootstrap.sh my-cluster
```

---

## 9. Monitoring

### Key Metrics

| Metric | Description |
| --- | --- |
| `karpenter_nodes_created_total` | Total nodes created |
| `karpenter_nodes_terminated_total` | Total nodes terminated |
| `karpenter_pods_pending_total` | Pods waiting for nodes |
| `karpenter_provisioning_duration_seconds` | Time to provision node |

### Grafana Dashboard

Import Karpenter dashboard: `https://grafana.com/grafana/dashboards/1860`

### Logging

```bash
# View Karpenter logs
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter

# Watch provisioning events
kubectl get events -A --field-selector reason=Provisioning
```

---

## 10. Troubleshooting

### Pods Stuck in Pending

```bash
# Check unschedulable pods
kubectl get pods -A --field-selector=status.phase=Pending -o wide

# Describe pod for constraints
kubectl describe pod <pod-name>

# Check Karpenter logs
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter | grep "cannot be scheduled"
```

### Nodes Not Provisioning

```bash
# Check NodePool status
kubectl get nodepool -o yaml

# Check EC2NodeClass status
kubectl get ec2nodeclass -o yaml

# Check Karpenter controller
kubectl get pods -n kube-system -l app.kubernetes.io/name=karpenter
```

### Spot Instance Issues

```bash
# Check spot capacity
aws ec2 describe-spot-instance-requests

# Review spot interruption history
kubectl get events -A --field-selector reason=SpotInterruption
```

---

## 11. Best Practices

### 1. Use Multiple NodePools

```yaml
# Critical workloads - on-demand
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: critical
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]

---
# Non-critical - spot
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
```

### 2. Set Resource Limits

Prevent runaway scaling:

```yaml
spec:
  limits:
    cpu: 1000
    memory: 1000Gi
```

### 3. Use Consolidation

Reduce costs by consolidating workloads:

```yaml
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```

### 4. Define Budgets

Control disruption rate:

```yaml
spec:
  disruption:
    budgets:
      - nodes: 10%
```

---

## 12. Provider-Specific Skills

For provider-specific configurations, see dedicated skills:

- **AWS EKS**: `karpenter-providers` 규칙의 EKS 섹션
- **Azure AKS**: `karpenter-providers` 규칙의 AKS 섹션
- **GCP GKE**: `karpenter-providers` 규칙의 GKE 섹션

---

## 13. Migration from Cluster Autoscaler

### Steps

1. Install Karpenter alongside Cluster Autoscaler
2. Create NodePools matching existing node groups
3. Gradually reduce Cluster Autoscaler node groups
4. Remove Cluster Autoscaler

### Migration Checklist

- [ ] Verify IAM permissions
- [ ] Create matching NodePools
- [ ] Test with non-production workloads
- [ ] Monitor cost comparison
- [ ] Remove Cluster Autoscaler

---

## 14. References

- [Karpenter Documentation](https://karpenter.sh/)
- [Karpenter GitHub](https://github.com/kubernetes-sigs/karpenter)
- [Karpenter Best Practices](https://karpenter.sh/docs/concepts/)
- [v1 Migration Guide](https://karpenter.sh/v1.0/upgrading/v1-migration/)
