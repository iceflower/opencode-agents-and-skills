# Managed Kubernetes Providers Guide

## 1. Provider Comparison

### Feature Matrix

| Feature | AWS EKS | Azure AKS | GCP GKE |
| --- | --- | --- | --- |
| Fully managed control plane | Yes | Yes | Yes |
| Serverless nodes | Fargate | - | Autopilot |
| Release channels | - | Auto-upgrade channels | Rapid / Regular / Stable / Extended |
| Advanced auto-scaler | Karpenter | - | Autopilot / NAP |
| Default CNI | AWS VPC CNI | Azure CNI | VPC-native (Calico) |
| Managed certificates | ACM + ALB | App Gateway | Managed Certificates (GKE) |
| Secret management | AWS Secrets Manager / SSM | Azure Key Vault CSI | Secret Manager |
| Workload identity | IRSA (OIDC) | Workload Identity (OIDC) | Workload Identity (GCP SA binding) |
| CLI tool | `eksctl` / `aws eks` | `az aks` | `gcloud container` |

### Version Support (2026-03-14)

| Provider | Active Versions | Support Policy |
| --- | --- | --- |
| EKS | 1.32 – 1.35 (Standard), 1.29 – 1.31 (Extended) | Standard ~14 months, Extended available at additional cost |
| AKS | 1.32 – 1.34 (GA), 1.31 (EOL May 2026) | 3 GA minor versions at any time |
| GKE | 1.33 – 1.35 (Active), 1.32 (EOL Apr 2026) | Channel-based: Rapid / Regular / Stable / Extended |

### Cost-Saving Options

| Strategy | EKS | AKS | GKE |
| --- | --- | --- | --- |
| Spot/preemptible nodes | Spot Instances, Karpenter | Spot VMs | Spot VMs, Preemptible VMs |
| ARM-based instances | Graviton (~20% cheaper) | - | Tau T2A |
| Serverless (pay-per-pod) | Fargate | - | Autopilot |
| Committed use discounts | Savings Plans / Reserved Instances | Azure Reservations | Committed Use Discounts (1-3 yr) |
| Cluster stop/start | - | `az aks stop/start` | - |
| License reuse | - | Azure Hybrid Benefit (Windows) | - |

---

## 2. Common Patterns

### Pod Security Standards

All providers support Kubernetes Pod Security Standards via namespace labels.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
```

### Network Policy (Default Deny)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Upgrade Best Practices

1. **Review release notes** and API deprecations for the target version
2. **Upgrade control plane** first
3. **Upgrade node pools** one at a time for HA
4. **Verify add-ons** compatibility after upgrade
5. **Run smoke tests** before routing production traffic

---

## 3. AWS EKS

### 3.1. Cluster Creation

#### eksctl (Recommended)

```bash
# Basic cluster
eksctl create cluster \
  --name production \
  --region us-east-1 \
  --version 1.35 \
  --nodegroup-name workers \
  --node-type m6i.xlarge \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 5 \
  --managed

# With Fargate profile
eksctl create cluster \
  --name serverless \
  --region us-east-1 \
  --version 1.35 \
  --fargate
```

#### EKS Terraform

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "production"
  cluster_version = "1.35"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  eks_managed_node_groups = {
    workers = {
      instance_types = ["m6i.xlarge"]
      min_size       = 1
      max_size       = 5
      desired_size   = 3
    }
  }

  # IRSA enabled by default
  enable_irsa = true
}
```

### 3.2. Node Groups

#### Managed Node Groups (Recommended)

```bash
eksctl create nodegroup \
  --cluster production \
  --name workers \
  --node-type m6i.xlarge \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 10 \
  --managed \
  --asg-access
```

#### Karpenter (Auto-scaling)

```bash
helm repo add karpenter https://charts.karpenter.sh
helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set cluster.name=production \
  --set cluster.endpoint=https://xxx.gr7.us-east-1.eks.amazonaws.com
```

```yaml
apiVersion: karpenter.sh/v1beta1
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot", "on-demand"]
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["m6i.xlarge", "m6i.2xlarge"]
  limits:
    resources:
      cpu: 1000
  providerRef:
    name: default
```

#### Fargate (Serverless)

```bash
eksctl create fargateprofile \
  --cluster production \
  --name serverless \
  --namespace serverless
```

### 3.3. Networking

#### AWS Load Balancer Controller

```bash
helm repo add eks-charts https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks-charts/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=production \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

#### Ingress (ALB)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 80
```

#### VPC CNI Configuration

| Setting | Default | Recommendation |
| --- | --- | --- |
| `WARM_ENI_TARGET` | 1 | 1-2 for production |
| `WARM_PREFIX_TARGET` | 0 | 1 for large clusters |
| `ENABLE_PREFIX_DELEGATION` | false | true for IP efficiency |

### 3.4. Add-ons

| Add-on | Purpose | Install |
| --- | --- | --- |
| `vpc-cni` | AWS VPC networking | Required |
| `coredns` | Cluster DNS | Required |
| `kube-proxy` | Service proxy | Required |
| `aws-ebs-csi-driver` | EBS persistent volumes | Recommended |
| `aws-efs-csi-driver` | EFS persistent volumes | Optional |
| `aws-load-balancer-controller` | ALB/NLB provisioning | Recommended |

```bash
eksctl create addon \
  --cluster production \
  --name aws-ebs-csi-driver \
  --version latest \
  --service-account-role-arn arn:aws:iam::123456789012:role/EBSCSIDriverRole
```

### 3.5. Storage

#### EBS CSI Driver

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
allowVolumeExpansion: true
```

#### EFS CSI Driver

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs
provisioner: efs.csi.aws.com
parameters:
  fileSystemId: fs-12345678
  directoryPerFS: "true"
```

### 3.6. Security

#### IAM Roles for Service Accounts (IRSA)

```bash
# Create OIDC provider
eksctl utils associate-iam-oidc-provider \
  --cluster production \
  --approve

# Create service account with IAM role
eksctl create iamserviceaccount \
  --cluster production \
  --name s3-access \
  --namespace default \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-access
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/S3AccessRole
```

#### Secrets Encryption

```bash
eksctl utils enable-kms-encryption \
  --cluster production \
  --key-arn arn:aws:kms:us-east-1:123456789012:key/xxx
```

### 3.7. Monitoring

#### CloudWatch Container Insights

```bash
aws eks put-cluster-logging \
  --cluster production \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
```

### 3.8. Upgrade Process

```bash
# 1. Check current version and deprecations
kubectl version
eksctl get addons --cluster production

# 2. Upgrade control plane
eksctl upgrade cluster \
  --name production \
  --version 1.35 \
  --approve

# 3. Upgrade add-ons
eksctl upgrade addon \
  --cluster production \
  --name vpc-cni \
  --version latest \
  --approve

# 4. Upgrade node groups
eksctl upgrade nodegroup \
  --cluster production \
  --name workers \
  --kubernetes-version 1.35

# 5. Check for upgrade issues
aws eks list-insights --cluster-name production
```

### 3.9. Troubleshooting

#### Node Not Ready

VPC CNI not running, IAM role missing, or security group blocking traffic.

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node
kubectl logs -n kube-system -l k8s-app=aws-node
```

#### IRSA Not Working

```bash
aws iam list-open-id-connect-providers
aws iam get-role --role-name EKSServiceAccountRole --query 'Role.AssumeRolePolicyDocument'
```

#### ALB Not Created

```bash
kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
# Verify subnets are tagged: kubernetes.io/cluster/<cluster-name>=shared
```

### 3.10. CLI Reference

```bash
eksctl get clusters
eksctl get nodegroups --cluster production
eksctl utils write-kubeconfig --cluster production
eksctl utils describe-stacks --cluster production
eksctl upgrade cluster --name production --version 1.35 --dry-run
```

### 3.11. References

- [EKS Documentation](https://docs.aws.amazon.com/eks/)
- [EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [eksctl Documentation](https://eksctl.io/)
- [Karpenter](https://karpenter.sh/)
- [EKS Kubernetes Versions](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)

---

## 4. Azure AKS

### 4.1. Cluster Creation

#### Azure CLI

```bash
# Basic cluster
az aks create \
  --resource-group myResourceGroup \
  --name production \
  --kubernetes-version 1.34 \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-managed-identity \
  --generate-ssh-keys

# With auto-scaling
az aks create \
  --resource-group myResourceGroup \
  --name production \
  --kubernetes-version 1.34 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10 \
  --node-vm-size Standard_D4s_v3
```

#### AKS Terraform

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "production"
  location            = azurerm_resource_group.example.location
  resource_group_name = azurerm_resource_group.example.name
  dns_prefix          = "production"
  kubernetes_version  = "1.34"

  default_node_pool {
    name       = "default"
    node_count = 3
    vm_size    = "Standard_D4s_v3"

    enable_auto_scaling = true
    min_count           = 1
    max_count           = 10
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin    = "azure"
    load_balancer_sku = "standard"
  }

  oms_agent {
    log_analytics_workspace_id = azurerm_log_analytics_workspace.workspace.id
  }
}
```

### 4.2. Node Pools

```bash
# System node pool
az aks nodepool add \
  --resource-group myResourceGroup \
  --cluster-name production \
  --name systempool \
  --node-count 2 \
  --node-vm-size Standard_D2s_v3 \
  --mode System

# User node pool
az aks nodepool add \
  --resource-group myResourceGroup \
  --cluster-name production \
  --name userpool \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --mode User \
  --labels workload=application

# Spot node pool
az aks nodepool add \
  --resource-group myResourceGroup \
  --cluster-name production \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --node-count 2 \
  --node-vm-size Standard_D4s_v3
```

### 4.3. Networking

#### CNI Options

| Plugin | Description |
| --- | --- |
| Azure CNI | Full VNet integration (default) |
| Kubenet | Simplified overlay networking |
| Azure CNI Overlay | Combines Azure CNI benefits with simplified IP management (recommended) |

```bash
# Azure CNI Overlay (recommended)
az aks create \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16
```

#### Network Policies

| Option | Description |
| --- | --- |
| `calico` | Open-source, feature-rich |
| `cilium` | eBPF-based, high performance |
| `azure` | Azure-native, simple |

#### Ingress Options

Application Gateway Ingress Controller (AGIC):

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    kubernetes.io/ingress.class: azure-application-gateway
    appgw.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
    - secretName: tls-secret
      hosts:
        - app.example.com
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app
                port:
                  number: 80
```

Managed NGINX:

```bash
az aks update \
  --resource-group myResourceGroup \
  --name production \
  --enable-web-application-routing
```

### 4.4. Add-ons

| Add-on | Purpose | Enable Command |
| --- | --- | --- |
| `azure-policy` | Governance | `--enable-azure-policy` |
| `azure-keyvault-secrets-provider` | Secrets management | `--enable-azure-keyvault-secrets-provider` |
| `open-service-mesh` | Service mesh | `--enable-open-service-mesh` |
| `ingress-appgw` | Application Gateway Ingress | `--enable-appgw-ingress` |
| `web_application_routing` | Managed NGINX ingress | `--enable-web-application-routing` |

```bash
az aks enable-addons \
  --resource-group myResourceGroup \
  --name production \
  --addons azure-keyvault-secrets-provider
```

### 4.5. Storage

#### Azure Disk (CSI)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
allowVolumeExpansion: true
```

#### Azure File (CSI)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile
provisioner: file.csi.azure.com
parameters:
  skuName: Standard_LRS
```

#### Blob Storage (CSI)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azureblob
provisioner: blob.csi.azure.com
parameters:
  skuName: Standard_LRS
```

### 4.6. Security

#### Managed Identity & Azure AD

```bash
# Managed identity (recommended)
az aks create --enable-managed-identity

# Grant access to ACR
az aks update \
  --resource-group myResourceGroup \
  --name production \
  --attach-acr myRegistry

# Azure AD integration
az aks create \
  --enable-aad \
  --aad-admin-group-object-ids <group-id>
```

#### AKS Workload Identity

```bash
az aks create \
  --enable-workload-identity \
  --enable-oidc-issuer
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workload-identity-sa
  annotations:
    azure.workload.identity/client-id: <client-id>
```

#### Azure Key Vault Provider

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault
spec:
  provider: azure
  parameters:
    keyvaultName: my-keyvault
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
    tenantId: <tenant-id>
```

#### Private Cluster & Authorized IP Ranges

```bash
az aks create \
  --resource-group myResourceGroup \
  --name production \
  --enable-private-cluster \
  --private-dns-zone system

az aks update \
  --resource-group myResourceGroup \
  --name production \
  --api-server-authorized-ip-ranges 10.0.0.0/8
```

### 4.7. Monitoring

```bash
# Azure Monitor Container Insights
az aks enable-addons \
  --resource-group myResourceGroup \
  --name production \
  --addons monitoring \
  --workspace-resource-id /subscriptions/.../logAnalyticsWorkspaces/myWorkspace

# Enable Prometheus & Grafana
az aks update \
  --resource-group myResourceGroup \
  --name production \
  --enable-azure-monitor-metrics
```

### 4.8. Upgrade Process

```bash
# Check available versions
az aks get-versions --location eastus --output table

# Upgrade control plane
az aks upgrade \
  --resource-group myResourceGroup \
  --name production \
  --kubernetes-version 1.34

# Upgrade node pools
az aks nodepool upgrade \
  --resource-group myResourceGroup \
  --cluster-name production \
  --name userpool \
  --kubernetes-version 1.34
```

#### Auto-upgrade Channels

| Channel | Description |
| --- | --- |
| `none` | No auto-upgrade |
| `patch` | Patch version updates only |
| `stable` | Minor version updates (recommended) |
| `rapid` | Latest versions quickly |

```bash
az aks update \
  --resource-group myResourceGroup \
  --name production \
  --auto-upgrade-channel stable
```

### 4.9. Troubleshooting

#### Node Pool Upgrade Failure

```bash
az aks nodepool show \
  --resource-group myResourceGroup \
  --cluster-name production \
  --name userpool

kubectl describe node <node-name>
```

#### DNS Resolution Issues

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get configmap coredns -n kube-system -o yaml
```

#### VNet IP Exhaustion

Use Azure CNI Overlay mode or increase subnet size.

```bash
az aks update \
  --resource-group myResourceGroup \
  --name production \
  --network-plugin azure \
  --network-plugin-mode overlay
```

### 4.10. CLI Reference

```bash
az aks get-credentials --resource-group myResourceGroup --name production
az aks list --output table
az aks show --resource-group myResourceGroup --name production
az aks nodepool list --cluster-name production --resource-group myResourceGroup
az aks stop --name production --resource-group myResourceGroup
az aks start --name production --resource-group myResourceGroup
```

### 4.11. References

- [AKS Documentation](https://learn.microsoft.com/azure/aks/)
- [AKS Best Practices](https://learn.microsoft.com/azure/aks/best-practices)
- [AKS Release Tracker](https://learn.microsoft.com/azure/aks/release-tracker)
- [AKS Supported Versions](https://learn.microsoft.com/azure/aks/supported-kubernetes-versions)

---

## 5. GCP GKE

### 5.1. Cluster Creation

#### gcloud CLI

```bash
# Standard cluster
gcloud container clusters create production \
  --region us-central1 \
  --release-channel regular \
  --num-nodes 3 \
  --machine-type e2-standard-4 \
  --enable-ip-alias

# Autopilot cluster (fully managed)
gcloud container clusters create-auto autopilot-cluster \
  --region us-central1
```

#### GKE Terraform

```hcl
resource "google_container_cluster" "primary" {
  name     = "production"
  location = "us-central1"

  release_channel {
    channel = "REGULAR"
  }

  initial_node_count = 1
  remove_default_node_pool = true

  network    = google_compute_network.vpc.name
  subnetwork = google_compute_subnetwork.subnet.name

  ip_allocation_policy {
    cluster_secondary_range_name  = "pods"
    services_secondary_range_name = "services"
  }

  private_cluster_config {
    enable_private_nodes    = true
    enable_private_endpoint = false
    master_ipv4_cidr_block  = "172.16.0.0/28"
  }
}

resource "google_container_node_pool" "primary_nodes" {
  name       = "primary-pool"
  cluster    = google_container_cluster.primary.name
  location   = "us-central1"

  node_count = 3

  node_config {
    machine_type = "e2-standard-4"
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }

  autoscaling {
    min_node_count = 1
    max_node_count = 10
  }

  management {
    auto_upgrade = true
    auto_repair  = true
  }
}
```

### 5.2. GKE Autopilot

Fully managed nodes, pay per pod resource request, auto-scaling built-in, security hardening by default.

```bash
gcloud container clusters create-auto autopilot-cluster \
  --region us-central1 \
  --release-channel regular
```

Autopilot pods require explicit resource requests and security context:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  template:
    spec:
      containers:
        - name: app
          image: app:latest
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
```

### 5.3. Node Pools

```bash
# Create node pool
gcloud container node-pools create workers \
  --cluster production \
  --region us-central1 \
  --num-nodes 3 \
  --machine-type e2-standard-4 \
  --disk-size 100GB \
  --disk-type pd-ssd \
  --enable-autoscaling \
  --min-nodes 1 \
  --max-nodes 10 \
  --node-version 1.34

# Spot VMs
gcloud container node-pools create spot-pool \
  --cluster production \
  --region us-central1 \
  --spot \
  --num-nodes 2
```

### 5.4. Networking

#### VPC-Native Clusters (Required)

```bash
gcloud container clusters create production \
  --enable-ip-alias \
  --cluster-secondary-range-name pods \
  --services-secondary-range-name services
```

#### Private Clusters & Authorized Networks

```bash
gcloud container clusters create private-cluster \
  --region us-central1 \
  --enable-private-nodes \
  --enable-private-endpoint \
  --master-ipv4-cidr 172.16.0.0/28 \
  --enable-ip-alias

gcloud container clusters update production \
  --region us-central1 \
  --enable-master-authorized-networks \
  --master-authorized-networks 10.0.0.0/8,192.168.0.0/16
```

#### Network Policies (Calico)

```bash
gcloud container clusters update production \
  --region us-central1 \
  --enable-network-policy
```

#### Release Channels & Maintenance Windows

```bash
# Update release channel
gcloud container clusters update production \
  --region us-central1 \
  --release-channel stable

# Maintenance window
gcloud container clusters update production \
  --region us-central1 \
  --maintenance-window-start "2026-03-15T02:00:00Z" \
  --maintenance-window-end "2026-03-15T06:00:00Z" \
  --maintenance-window-recurrence "FREQ=WEEKLY;BYDAY=SA,SU"
```

### 5.5. Ingress & Load Balancing

#### GKE Ingress (HTTP(S) Load Balancer)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    kubernetes.io/ingress.class: "gce"
    kubernetes.io/ingress.global-static-ip-name: "app-ip"
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: app
                port:
                  number: 80
```

#### Internal Load Balancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-internal
  annotations:
    cloud.google.com/load-balancer-type: "Internal"
spec:
  type: LoadBalancer
  loadBalancerIP: 10.0.0.100
  ports:
    - port: 80
  selector:
    app: app
```

#### Managed Certificates

```yaml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: app-cert
spec:
  domains:
    - app.example.com
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app
  annotations:
    networking.gke.io/managed-certificates: "app-cert"
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /*
            pathType: ImplementationSpecific
            backend:
              service:
                name: app
                port:
                  number: 80
```

### 5.6. Storage

#### Persistent Disk (GCE PD)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

#### Filestore

```bash
gcloud container clusters update production \
  --region us-central1 \
  --update-addons FilestoreCSIDriver=ENABLED
```

### 5.7. Security

#### GKE Workload Identity

```bash
gcloud container clusters update production \
  --region us-central1 \
  --workload-pool=PROJECT_ID.svc.id.goog
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gcs-access
  annotations:
    iam.gke.io/gcp-service-account: gcs-reader@PROJECT_ID.iam.gserviceaccount.com
```

```bash
gcloud iam service-accounts add-iam-policy-binding \
  gcs-reader@PROJECT_ID.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[default/gcs-access]"
```

#### Binary Authorization

```bash
gcloud container clusters update production \
  --region us-central1 \
  --binauthz-evaluation-mode PROJECT_SINGLETON_POLICY_ENFORCE
```

#### Confidential Nodes

```bash
gcloud container node-pools create confidential-pool \
  --cluster production \
  --region us-central1 \
  --confidential-nodes \
  --machine-type n2d-standard-4
```

### 5.8. Monitoring

```bash
# Configure logging and monitoring
gcloud container clusters update production \
  --region us-central1 \
  --logging=SYSTEM,WORKLOAD \
  --monitoring=SYSTEM,WORKLOAD

# Enable managed Prometheus
gcloud container clusters update production \
  --region us-central1 \
  --enable-managed-prometheus

# Multi-cluster management (Fleet)
gcloud container fleet memberships register production \
  --gke-cluster us-central1/production \
  --enable-workload-identity
```

### 5.9. Upgrade Process

```bash
# Upgrade control plane
gcloud container clusters upgrade production \
  --region us-central1 \
  --cluster-version 1.35

# Upgrade specific node pool
gcloud container node-pools upgrade primary-pool \
  --cluster production \
  --region us-central1
```

#### Surge Upgrades

```bash
gcloud container node-pools update primary-pool \
  --cluster production \
  --region us-central1 \
  --max-surge 3 \
  --max-unavailable 1
```

### 5.10. Troubleshooting

#### Image Pull Permission Denied

```bash
gcloud artifacts repositories add-iam-policy-binding my-repo \
  --location us-central1 \
  --member "serviceAccount:PROJECT-compute@developer.gserviceaccount.com" \
  --role roles/artifactregistry.reader
```

#### Insufficient Resources (Pods Pending)

```bash
kubectl describe nodes

gcloud container clusters resize production \
  --region us-central1 \
  --node-pool primary-pool \
  --num-nodes 5
```

#### Private Cluster Access

```bash
gcloud container clusters update production \
  --region us-central1 \
  --enable-master-authorized-networks \
  --master-authorized-networks YOUR_IP/32
```

### 5.11. CLI Reference

```bash
gcloud container clusters get-credentials production --region us-central1
gcloud container clusters list
gcloud container clusters describe production --region us-central1
gcloud container node-pools list --cluster production --region us-central1
gcloud container operations list --region us-central1
```

### 5.12. References

- [GKE Documentation](https://cloud.google.com/kubernetes-engine/docs)
- [GKE Best Practices](https://cloud.google.com/kubernetes-engine/docs/best-practices)
- [GKE Release Notes](https://cloud.google.com/kubernetes-engine/docs/release-notes)
- [GKE Versioning](https://cloud.google.com/kubernetes-engine/versioning)
- [GKE Release Schedule](https://cloud.google.com/kubernetes-engine/docs/release-schedule)
