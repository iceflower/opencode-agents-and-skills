
# Karpenter Cloud Providers Guide

## 1. Provider Comparison

| Feature | AWS EKS | Azure AKS | GCP GKE |
| --- | --- | --- | --- |
| Status | GA | Preview | Experimental |
| NodeClass API | `EC2NodeClass` (v1) | `AKSNodeClass` (v1alpha1) | `GKENodeClass` (v1alpha1) |
| API Group | `karpenter.k8s.aws` | `karpenter.azure.com` | `karpenter.gke.com` |
| Spot/Preemptible | EC2 Spot | Azure Spot VM | GCE Spot VM |
| Identity | IRSA / EKS Pod Identity | Managed Identity | Workload Identity |
| Interruption Handling | SQS + EventBridge | Spot Eviction Policy | Preemption Events |
| Native Alternative | Cluster Autoscaler | AKS Cluster Autoscaler | Autopilot / Node Auto-provisioning |

### When to Use Karpenter

- Dynamic workload patterns requiring flexible instance selection
- Spot/preemptible instance optimization
- Multi-cloud consistency with unified tooling
- Advanced scheduling and consolidation requirements

---

## 2. Common Installation

### Helm Installation (All Providers)

```bash
# Add Helm repo
helm repo add karpenter https://charts.karpenter.sh
helm repo update

# Install Karpenter (adjust settings per provider)
helm upgrade --install karpenter karpenter/karpenter \
  --namespace kube-system \
  --set settings.clusterName=my-cluster \
  --set settings.clusterEndpoint=<CLUSTER_ENDPOINT> \
  --version v1.0.5
```

### Upgrade

```bash
helm upgrade karpenter karpenter/karpenter \
  --namespace kube-system \
  --version v1.0.5
```

### Upgrade Checklist

1. Review [compatibility matrix](https://karpenter.sh/docs/upgrading/compatibility/)
2. Check Kubernetes version support
3. Backup NodePool configurations
4. Upgrade Helm chart
5. Verify node provisioning

---

## 3. Common Monitoring and Metrics

### Prometheus Integration

```yaml
apiVersion: v1
kind: Service
metadata:
  name: karpenter-metrics
  namespace: kube-system
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app.kubernetes.io/name: karpenter
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: karpenter
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: karpenter
  endpoints:
    - port: metrics
```

### Provider-Specific Metrics

| Provider | Metric | Description |
| --- | --- | --- |
| AWS | (core metrics) | Standard Karpenter metrics |
| Azure | `karpenter_azure_vm_available` | Available VMs |
| Azure | `karpenter_azure_spot_eviction` | Spot evictions |
| GCP | `karpenter_gce_instances_total` | GCE instances managed |
| GCP | `karpenter_gce_spot_preemption` | Spot preemption events |

### Common Debug Commands

```bash
# List all nodes created by Karpenter
kubectl get nodes -l karpenter.sh/nodepool

# Check NodePool status
kubectl get nodepool -o yaml

# View provisioning events
kubectl get events -A --field-selector reason=Provisioning

# Check Karpenter logs
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter
```

### Common Consolidation Settings

```yaml
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
    budgets:
      - nodes: 10%
```

---

## 4. AWS EKS Provider

### 4.1. IAM Setup

```bash
# Create Karpenter IAM role
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --name karpenter \
  --namespace kube-system \
  --role-name KarpenterControllerRole \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy \
  --approve

# Create node IAM role
aws iam create-role \
  --role-name KarpenterNodeRole \
  --assume-role-policy-document file://trust-policy.json

# Attach policies
aws iam attach-role-policy \
  --role-name KarpenterNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy

aws iam attach-role-policy \
  --role-name KarpenterNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy

aws iam attach-role-policy \
  --role-name KarpenterNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam attach-role-policy \
  --role-name KarpenterNodeRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

### EKS-Specific Helm Settings

```bash
helm upgrade --install karpenter karpenter/karpenter \
  --namespace kube-system \
  --set serviceAccount.create=false \
  --set serviceAccount.name=karpenter \
  --set settings.clusterName=my-cluster \
  --set settings.clusterEndpoint=https://xxx.gr7.us-east-1.eks.amazonaws.com \
  --set settings.interruptionQueue=my-cluster \
  --version v1.0.5
```

### 4.2. EC2NodeClass Configuration

#### EKS Basic Configuration

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
```

#### AMI Selection

| Alias | Description |
| --- | --- |
| `al2023@latest` | Amazon Linux 2023 (recommended) |
| `al2@latest` | Amazon Linux 2 |
| `ubuntu@latest` | Ubuntu 22.04 |
| `bottlerocket@latest` | Bottlerocket OS |

```yaml
spec:
  amiSelectorTerms:
    # AL2023
    - alias: al2023@latest

    # Custom AMI
    - id: ami-12345678

    # By tags
    - tags:
        Name: my-custom-ami
```

#### Subnet Selection

```yaml
spec:
  subnetSelectorTerms:
    # By tags
    - tags:
        karpenter.sh/discovery: my-cluster

    # By ID
    - id: subnet-12345

    # Filter by type
    - tags:
        Type: private
```

#### Security Groups

```yaml
spec:
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: my-cluster

    # Additional security groups
    - id: sg-12345
```

#### Custom Tags

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  tags:
    Environment: production
    Team: platform
    CostCenter: engineering
```

#### User Data

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  userData: |
    #!/bin/bash
    /etc/eks/bootstrap.sh my-cluster \
      --kubelet-extra-args '--node-labels=karpenter.sh/capacity-type=spot' \
      --b64-extra-ca-certificates dGVzdA==
```

#### Bottlerocket Configuration

```yaml
spec:
  amiSelectorTerms:
    - alias: bottlerocket@latest
  userData: |
    [settings.kubernetes]
    cluster-name = "my-cluster"
    api-server = "https://xxx.gr7.us-east-1.eks.amazonaws.com"
```

#### Block Device Configuration

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        encrypted: true
        deleteOnTermination: true
```

### 4.3. Instance Types

#### Instance Requirements

```yaml
spec:
  template:
    spec:
      requirements:
        # Instance categories
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]

        # Generations
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["5"]

        # Specific families
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m6i", "m7i", "c6i", "c7i"]

        # Exclude sizes
        - key: karpenter.k8s.aws/instance-size
          operator: NotIn
          values: ["nano", "micro", "small"]
```

#### Available Labels

| Label | Example Values |
| --- | --- |
| `karpenter.k8s.aws/instance-category` | c, m, r, t, x, i, d |
| `karpenter.k8s.aws/instance-family` | m6i, c7i, r6a |
| `karpenter.k8s.aws/instance-generation` | 5, 6, 7 |
| `karpenter.k8s.aws/instance-size` | xlarge, 2xlarge, 4xlarge |
| `karpenter.k8s.aws/instance-cpu` | 2, 4, 8, 16 |
| `karpenter.k8s.aws/instance-memory` | 8192, 16384, 32768 |
| `karpenter.k8s.aws/instance-hypervisor` | nitro, xen |
| `karpenter.k8s.aws/instance-local-nvme` | 0, 1, 2 |

#### GPU Instances

```yaml
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["g5", "g6", "p4", "p5"]
```

### 4.4. Spot Instances

#### EKS Spot Configuration

```yaml
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
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m6i", "m7i", "c6i", "c7i", "r6i", "r7i"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  disruption:
    consolidateAfter: 1m
```

#### Spot Interruption Queue

```bash
# Create SQS queue for spot interruptions
aws sqs create-queue \
  --queue-name my-cluster \
  --attributes MessageRetentionPeriod=300

# Configure EventBridge rules
aws events put-rule \
  --name KarpenterSpotInterruption \
  --event-pattern '{"source":["aws.ec2"],"detail-type":["EC2 Spot Instance Interruption Warning"]}'
```

### 4.5. EKS-Specific Features

#### IRSA Integration

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  role: KarpenterNodeRole
```

Karpenter v1.0+ also supports EKS Pod Identity for workloads.

#### EKS Add-ons Compatibility

| Add-on | Karpenter Compatible |
| --- | --- |
| VPC CNI | Yes |
| CoreDNS | Yes |
| kube-proxy | Yes |
| EBS CSI Driver | Yes |
| EFS CSI Driver | Yes |
| AWS Load Balancer Controller | Yes |

### 4.6. Cost Optimization

#### Spot vs On-Demand

```yaml
# Spot for non-critical
---
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
  weight: 100  # Prefer spot

---
# On-demand for critical
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: on-demand
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
  weight: 50
```

#### Graviton (ARM) Instances

```yaml
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["arm64"]
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m7g", "c7g", "r7g"]  # ~20% cheaper
```

### 4.7. EKS Troubleshooting

#### Instance Launch Failures

```bash
# Check Karpenter logs
kubectl logs -n kube-system -l app.kubernetes.io/name=karpenter | grep "InstanceLaunch"

# Check EC2 limits
aws service-quotas list-service-quotas --service-code ec2
```

#### Subnet Errors

```bash
# Verify subnet tags
aws ec2 describe-subnets --subnet-ids subnet-xxx --query 'Subnets[*].Tags'

# Check available IPs
aws ec2 describe-subnets --subnet-ids subnet-xxx --query 'Subnets[*].AvailableIpAddressCount'
```

#### IAM Permission Errors

```bash
# Verify role trust policy
aws iam get-role --role-name KarpenterNodeRole --query 'Role.AssumeRolePolicyDocument'

# Check attached policies
aws iam list-attached-role-policies --role-name KarpenterNodeRole
```

#### Spot Interruptions

```bash
# Check spot interruptions
aws sqs receive-message --queue-url https://sqs.us-east-1.amazonaws.com/xxx/my-cluster
```

### 4.8. EKS References

- [Karpenter AWS Provider Docs](https://karpenter.sh/docs/cloud-provider/aws/)
- [EC2NodeClass API](https://karpenter.sh/docs/reference/cloud-provider/aws/ec2nodeclass/)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/karpenter/)
- [Spot Instance Advisor](https://aws.amazon.com/ec2/spot/instance-advisor/)

---

## 5. Azure AKS Provider

> **Preview**: Not recommended for production workloads.
> GitHub: [Azure/karpenter-provider-azure](https://github.com/Azure/karpenter-provider-azure)

### 5.1. Prerequisites and Installation

```bash
# Install aks-preview extension
az extension add --name aks-preview
az extension update --name aks-preview

# Register feature
az feature register --namespace Microsoft.ContainerService --name KarpenterPreview

# Create AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name my-cluster \
  --kubernetes-version 1.31 \
  --enable-managed-identity \
  --generate-ssh-keys
```

#### AKS-Specific Helm Settings

```bash
helm upgrade --install karpenter karpenter/karpenter \
  --namespace kube-system \
  --set settings.clusterName=my-cluster \
  --set settings.clusterEndpoint=https://xxx.hcp.eastus.azmk8s.io:443 \
  --version v1.0.5
```

### 5.2. AKSNodeClass Configuration

#### AKS Basic Configuration

```yaml
apiVersion: karpenter.azure.com/v1alpha1
kind: AKSNodeClass
metadata:
  name: default
spec:
  imageFamily: Ubuntu2204
  osDiskSizeGB: 128
  osDiskType: Managed
```

#### Image Selection

| Image Family | Description |
| --- | --- |
| `Ubuntu2204` | Ubuntu 22.04 LTS |
| `Ubuntu2004` | Ubuntu 20.04 LTS |
| `AzureLinux` | Azure Linux (Mariner) |
| `Windows2022` | Windows Server 2022 |

#### Storage Configuration

```yaml
apiVersion: karpenter.azure.com/v1alpha1
kind: AKSNodeClass
metadata:
  name: default
spec:
  osDiskSizeGB: 256
  osDiskType: Managed  # or Ephemeral
  osDiskSku: Premium_LRS
```

| Disk Type | Performance | Cost |
| --- | --- | --- |
| `Standard_LRS` | Baseline | Low |
| `StandardSSD_LRS` | Moderate | Medium |
| `Premium_LRS` | High | High |
| `PremiumV2_LRS` | Highest | Highest |

#### AKS Network Configuration

```yaml
apiVersion: karpenter.azure.com/v1alpha1
kind: AKSNodeClass
metadata:
  name: default
spec:
  subnetId: /subscriptions/xxx/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/myVNet/subnets/mySubnet
```

### 5.3. VM Types and Labels

| Category | VM Sizes |
| --- | --- |
| General Purpose | D-series (D4s_v5, D8s_v5) |
| Compute Optimized | F-series (F4s_v2, F8s_v2) |
| Memory Optimized | E-series (E4s_v5, E8s_v5) |
| GPU | NC-series, NV-series |
| Spot | All sizes with spot support |

| Label | Description |
| --- | --- |
| `karpenter.azure.com/vm-size` | Azure VM size |
| `karpenter.azure.com/vm-family` | D, E, F, etc. |
| `karpenter.azure.com/vm-generation` | V4, V5 |
| `karpenter.azure.com/vm-cpu` | vCPU count |
| `karpenter.azure.com/vm-memory` | Memory in GB |

#### VM Requirements Example

```yaml
spec:
  template:
    spec:
      requirements:
        - key: karpenter.azure.com/vm-family
          operator: In
          values: ["D", "E"]

        - key: karpenter.azure.com/vm-generation
          operator: Gt
          values: ["V4"]

        - key: karpenter.azure.com/vm-cpu
          operator: In
          values: ["4", "8", "16"]
```

### 5.4. Spot VMs

#### AKS Spot Configuration

```yaml
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
        - key: karpenter.azure.com/vm-size
          operator: In
          values:
            - Standard_D4s_v5
            - Standard_D8s_v5
      nodeClassRef:
        group: karpenter.azure.com
        kind: AKSNodeClass
        name: default
```

Azure Spot VMs can be up to 90% cheaper than pay-as-you-go.

#### Spot Eviction Policy

```yaml
apiVersion: karpenter.azure.com/v1alpha1
kind: AKSNodeClass
metadata:
  name: spot
spec:
  spotEvictionPolicy: Delete
```

### 5.5. AKS Integration Features

#### Managed Identity

```yaml
apiVersion: karpenter.azure.com/v1alpha1
kind: AKSNodeClass
metadata:
  name: default
spec:
  managedIdentityId: /subscriptions/xxx/resourcegroups/myRG/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myIdentity
```

#### Azure Monitor

```yaml
apiVersion: karpenter.azure.com/v1alpha1
kind: AKSNodeClass
metadata:
  name: default
spec:
  enableAzureMonitor: true
```

```bash
# Enable Container Insights
az aks enable-addons \
  --resource-group myResourceGroup \
  --name my-cluster \
  --addons monitoring
```

### 5.6. NodePool Examples

#### Production NodePool

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: production
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: karpenter.azure.com/vm-family
          operator: In
          values: ["D"]
        - key: karpenter.azure.com/vm-generation
          operator: Gt
          values: ["V4"]
      nodeClassRef:
        group: karpenter.azure.com
        kind: AKSNodeClass
        name: production
  limits:
    cpu: 1000
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
---
apiVersion: karpenter.azure.com/v1alpha1
kind: AKSNodeClass
metadata:
  name: production
spec:
  imageFamily: Ubuntu2204
  osDiskSizeGB: 256
  osDiskType: Managed
  osDiskSku: Premium_LRS
```

### 5.7. Comparison with AKS Cluster Autoscaler

| Feature | Karpenter | AKS Cluster Autoscaler |
| --- | --- | --- |
| Scaling model | Pod-driven | Node pool-driven |
| Instance selection | Dynamic | Pre-configured pools |
| Spot support | Native | Limited |
| Consolidation | Active | Passive |
| Status | Preview | GA |

- Use **Karpenter** for dynamic workloads, spot optimization, multi-VM-size workloads, cost-sensitive environments
- Use **Cluster Autoscaler** for production stability, simpler requirements, Windows node pools, official AKS support

### 5.8. Limitations (Preview)

- Not production-ready: use for testing only
- Limited VM types: not all Azure VM sizes supported
- No Windows support: Linux nodes only
- No GPU scheduling: limited GPU support
- Regional availability: not available in all regions

### 5.9. AKS Troubleshooting

#### VM Unavailable

```bash
az vm list-sizes --location eastus -o table
```

#### Subnet Full

```bash
az network vnet subnet show \
  --resource-group myRG \
  --vnet-name myVNet \
  --name mySubnet \
  --query "addressPrefix"
```

#### Quota Limits

```bash
az vm list-usage --location eastus -o table
```

### 5.10. AKS References

- [Karpenter Azure Provider](https://github.com/Azure/karpenter-provider-azure)
- [AKS Documentation](https://learn.microsoft.com/azure/aks/)
- [Azure VM Sizes](https://learn.microsoft.com/azure/virtual-machines/sizes)
- [Spot VMs](https://learn.microsoft.com/azure/virtual-machines/spot-vms)

---

## 6. GCP GKE Provider

> **Experimental**: Not officially supported by Google. Community-driven support.
> GKE Standard cluster required (not Autopilot).

### 6.1. GKE Autopilot vs Karpenter

GKE Autopilot provides fully managed Kubernetes nodes with automatic provisioning, pay-per-pod pricing, built-in security hardening, and no node management overhead.

| Feature | Autopilot | Karpenter |
| --- | --- | --- |
| Management | Fully managed | Self-managed |
| Pricing | Per pod | Per node |
| Node visibility | Abstracted | Full access |
| Customization | Limited | Full control |
| Spot/Preemptible | Supported | Supported |

#### Recommendation

For GKE workloads, prefer **Autopilot** unless you have specific requirements:

- Use **Autopilot** for zero node management, pay-per-use model, security hardening
- Use **Karpenter** for node-level control, multi-cloud consistency, advanced scheduling
- Use **GKE Cluster Autoscaler** for stable production-ready autoscaling with Google-managed components

### 6.2. GKE Native Options

| Option | Description | Karpenter Alternative |
| --- | --- | --- |
| **Autopilot** | Fully managed nodes | Not needed |
| **Cluster Autoscaler** | Traditional node pool scaling | Karpenter replaces |
| **Node auto-provisioning** | Automatic node pool creation | Similar to Karpenter |

#### Node Auto-Provisioning

```bash
gcloud container clusters update my-cluster \
  --region us-central1 \
  --enable-autoprovisioning \
  --min-cpu 1 \
  --max-cpu 100 \
  --min-memory 1 \
  --max-memory 400
```

| Feature | Karpenter | GKE Auto-provisioning |
| --- | --- | --- |
| Pod-driven scaling | Yes | Yes |
| Spot/Preemptible | Yes | Yes |
| Custom scheduling | Full control | Limited |
| Consolidation | Active | Passive |
| Multi-cloud | Yes | No (GKE only) |
| Management | Self-managed | Google-managed |

### 6.3. Prerequisites and Installation

```bash
# Create GKE cluster
gcloud container clusters create my-cluster \
  --region us-central1 \
  --enable-ip-alias \
  --workload-pool=PROJECT_ID.svc.id.goog
```

#### GKE-Specific Helm Settings

```bash
helm upgrade --install karpenter karpenter/karpenter \
  --namespace kube-system \
  --set settings.clusterName=my-cluster \
  --set settings.clusterEndpoint=https://xxx.us-central1.containers.googleapis.com \
  --version v1.0.5
```

### 6.4. GKENodeClass Configuration

#### GKE Basic Configuration

```yaml
apiVersion: karpenter.gke.com/v1alpha1
kind: GKENodeClass
metadata:
  name: default
spec:
  machineType: e2-standard-4
  diskSizeGb: 100
  diskType: pd-ssd
  imageType: COS_CONTAINERD
```

#### Node Images

| Image Type | Description |
| --- | --- |
| `COS_CONTAINERD` | Container-Optimized OS (recommended) |
| `UBUNTU_CONTAINERD` | Ubuntu with containerd |
| `UBUNTU` | Ubuntu with Docker (deprecated) |

#### Shielded GKE Nodes

```yaml
spec:
  shieldedInstanceConfig:
    enableSecureBoot: true
    enableIntegrityMonitoring: true
```

#### GKE Network Configuration

```yaml
apiVersion: karpenter.gke.com/v1alpha1
kind: GKENodeClass
metadata:
  name: default
spec:
  subnetwork: projects/PROJECT/regions/us-central1/subnetworks/my-subnet
  serviceAccount: my-sa@PROJECT.iam.gserviceaccount.com
```

### 6.5. Machine Types and Labels

| Category | Types |
| --- | --- |
| General Purpose | e2-standard, n2-standard, n2d-standard |
| Compute Optimized | c2-standard, c2d-standard |
| Memory Optimized | m2-standard, m3-standard |
| Spot | All types available as Spot |

| Label | Description |
| --- | --- |
| `karpenter.gke.com/machine-type` | GCE machine type |
| `karpenter.gke.com/machine-family` | e2, n2, n2d, c2 |
| `karpenter.gke.com/machine-cpu` | vCPU count |
| `karpenter.gke.com/machine-memory` | Memory in GB |

#### Machine Type Requirements Example

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: karpenter.gke.com/machine-family
          operator: In
          values: ["n2", "n2d"]

        - key: karpenter.gke.com/machine-cpu
          operator: In
          values: ["4", "8", "16"]

        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
      nodeClassRef:
        group: karpenter.gke.com
        kind: GKENodeClass
        name: default
```

### 6.6. Spot/Preemptible VMs

#### GKE Spot Configuration

```yaml
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
        - key: karpenter.gke.com/machine-type
          operator: In
          values:
            - n2-standard-4
            - n2-standard-8
            - n2d-standard-4
      nodeClassRef:
        group: karpenter.gke.com
        kind: GKENodeClass
        name: spot
```

#### GKE Native Alternative

```bash
gcloud container node-pools create spot-pool \
  --cluster my-cluster \
  --region us-central1 \
  --spot \
  --num-nodes 2
```

### 6.7. Workload Identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: karpenter
  namespace: kube-system
  annotations:
    iam.gke.io/gcp-service-account: karpenter@PROJECT.iam.gserviceaccount.com
```

### 6.8. Cost Comparison

| Model | Pricing | Management |
| --- | --- | --- |
| Autopilot | Pay per pod resource request | Automatic scaling included |
| Karpenter + Standard GKE | Pay per node (VM pricing) | Node management required |

### 6.9. Limitations

- Experimental status: not officially supported by Google
- Limited documentation: community-driven support
- GKE Autopilot incompatibility: cannot use on Autopilot clusters
- Feature gaps: some GKE features not supported

### 6.10. GKE Troubleshooting

#### Machine Type Unavailable

```bash
gcloud compute machine-types list --zones us-central1-a
```

#### Quota Exceeded

```bash
gcloud compute project-info describe --project PROJECT_ID
```

#### Preemption Events

```bash
gcloud compute operations list --filter="operationType=compute.instances.preempted"
```

#### GKE-Specific Debug

```bash
# GCE instances
gcloud compute instances list --filter="metadata.cluster-name=my-cluster"

# Enable GKE monitoring
gcloud container clusters update my-cluster \
  --region us-central1 \
  --enable-stackdriver-kubernetes
```

### 6.11. GKE References

- [GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)
- [GKE Autopilot](https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview)
- [GKE Node Auto-provisioning](https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-provisioning)
- [Spot VMs](https://cloud.google.com/compute/docs/instances/spot)
