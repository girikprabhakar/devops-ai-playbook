# Infrastructure Summary

This document describes every Terraform resource in `projects/Infrastructure/`, how each one is wired to the others, and why it exists. The stack deploys a production-grade Kubernetes platform on AWS using EKS, with GitOps delivery via ArgoCD and observability via Prometheus/Grafana.

---

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Resource Connection Map (ASCII)](#resource-connection-map-ascii)
3. [Root Configuration](#root-configuration)
4. [Module: VPC](#module-vpc)
5. [Module: EKS](#module-eks)
6. [Module: ECR](#module-ecr)
7. [Module: ArgoCD](#module-argocd)
8. [Data Flow Between Modules](#data-flow-between-modules)
9. [Actual Values (terraform.tfvars)](#actual-values-terraformtfvars)
10. [Outputs Exposed to the Caller](#outputs-exposed-to-the-caller)

---

## High-Level Architecture

```
 ┌─────────────────── AWS Account (us-east-1) ─────────────────────────┐
 │                                                                       │
 │  ┌──────────────── VPC: 10.1.0.0/16 ──────────────────────────────┐  │
 │  │                                                                  │  │
 │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │  │
 │  │  │  Subnet-1    │  │  Subnet-2    │  │  Subnet-3    │          │  │
 │  │  │ 10.1.1.0/24  │  │ 10.1.2.0/24  │  │ 10.1.3.0/24  │          │  │
 │  │  │  us-east-1a  │  │  us-east-1b  │  │  us-east-1c  │          │  │
 │  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │  │
 │  │         └─────────────────┼─────────────────┘                   │  │
 │  │                    Route Table (0.0.0.0/0 → IGW)                │  │
 │  │                           │                                      │  │
 │  │                  Internet Gateway (IGW)                          │  │
 │  │                           │                                      │  │
 │  │         ┌─────────────────▼──────────────────────┐              │  │
 │  │         │          EKS Control Plane              │              │  │
 │  │         │         (eks-cluster v1.34)             │              │  │
 │  │         │       Public endpoint enabled           │              │  │
 │  │         └──────────────┬─────────────────────────┘              │  │
 │  │                        │                                         │  │
 │  │         ┌──────────────▼──────────────────────────┐             │  │
 │  │         │        EKS Managed Node Group           │             │  │
 │  │         │   1-2 × m7i-flex.large (ON_DEMAND)      │             │  │
 │  │         │   30 GB disk, spread across 3 AZs       │             │  │
 │  │         └──────────────┬──────────────────────────┘             │  │
 │  │                        │                                         │  │
 │  │         ┌──────────────┴──────────────────────────┐             │  │
 │  │         │          Kubernetes Workloads            │             │  │
 │  │         │  ┌──────────────┐  ┌─────────────────┐  │             │  │
 │  │         │  │  ns: argocd  │  │  ns: monitoring  │  │            │  │
 │  │         │  │  ArgoCD      │  │  Prometheus      │  │            │  │
 │  │         │  │  (GitOps)    │  │  Grafana         │  │            │  │
 │  │         │  │              │  │  Alertmanager     │  │            │  │
 │  │         │  └──────────────┘  └─────────────────┘  │            │  │
 │  │         └─────────────────────────────────────────┘             │  │
 │  └──────────────────────────────────────────────────────────────────┘  │
 │                                                                         │
 │  ┌───────────────────── ECR Repositories ────────────────────────────┐ │
 │  │  frontend | gateway | auth | order-service | orders               │ │
 │  │  product-service | user-service                                   │ │
 │  └───────────────────────────────────────────────────────────────────┘ │
 └─────────────────────────────────────────────────────────────────────────┘
```

---

## Resource Connection Map (ASCII)

This shows the exact Terraform dependency graph — arrows mean "depends on" or "receives output from".

```
terraform.tfvars
      │  (provides all input values)
      ▼
  variables.tf
      │
      ▼
  main.tf  ──────────────────────────────────────────────────────────┐
      │                                                               │
      │ module.vpc                                                    │
      │   aws_vpc ─────────────────────────────────┐                 │
      │     │ vpc_id                               │                 │
      │     ▼                                      │                 │
      │   aws_internet_gateway (igw)               │                 │
      │     │ gateway_id                           │                 │
      │     ▼                                      │                 │
      │   aws_route_table.public                   │ vpc_id          │
      │     │ route_table_id                       │                 │
      │     ▼                                      ▼                 │
      │   aws_route_table_association[0,1,2] ← aws_subnet[0,1,2]    │
      │                                            │                 │
      │                                  OUTPUT: subnet_ids          │
      │                                            │                 │
      ├────────────────────────────────────────────┘                 │
      │ module.eks  (depends_on: module.vpc)                         │
      │   INPUT: subnet_ids ← module.vpc.subnet_ids                  │
      │                                                              │
      │   aws_iam_role.eks_cluster_role                              │
      │     │ role (name)                                            │
      │     ▼                                                        │
      │   aws_iam_role_policy_attachment.cluster_policy              │
      │     │ (depends_on)                                           │
      │     └──────────────────────────────────────┐                │
      │   aws_iam_role.eks_cluster_role             │                │
      │     │ role_arn                              │                │
      │     ▼                                       ▼                │
      │   aws_eks_cluster.eks ◄────────────────────┘ (cluster_policy│
      │     │ satisfied first)                                       │
      │     ├──► identity[0].oidc[0].issuer                         │
      │     │         │                                              │
      │     │         ▼                                              │
      │     │   data.tls_certificate.eks                             │
      │     │         │ sha1_fingerprint + url                       │
      │     │         ▼                                              │
      │     │   aws_iam_openid_connect_provider.eks                  │
      │     │         │ arn                                          │
      │     │         └────────────────────────────────────┐         │
      │     │                                              │         │
      │     ├── cluster_name ──────────────────────────────────────┐ │
      │     │                                              │       │ │
      │     │   (EBS CSI IRSA path)                        │       │ │
      │     │   data.aws_iam_policy_document.ebs_csi ◄────┘       │ │
      │     │       (OIDC issuer condition)                        │ │
      │     │         │ json                                       │ │
      │     │         ▼                                            │ │
      │     │   aws_iam_role.ebs_csi_irsa                         │ │
      │     │         │ role (name)                                │ │
      │     │         ▼                                            │ │
      │     │   aws_iam_role_policy_attachment.ebs_csi_irsa_policy │ │
      │     │         │ (depends_on) + role_arn from ebs_csi_irsa  │ │
      │     │         ▼                ◄────────────────────────── │─┘
      │     │   aws_eks_addon.ebs_csi (cluster_name from eks)      │
      │     │                                                       │
      │     │   (Node Group path)                                   │
      │   aws_iam_role.eks_node_role                                │
      │     │ role (name)                                           │
      │     ├──► aws_iam_role_policy_attachment.worker_node_policy  │
      │     ├──► aws_iam_role_policy_attachment.cni_policy          │
      │     └──► aws_iam_role_policy_attachment.ecr_policy          │
      │               │ all 3 must complete first (depends_on)       │
      │               ▼                                              │
      │   aws_eks_node_group.node_group                              │
      │     ├── cluster_name ← aws_eks_cluster.eks.name             │
      │     ├── node_role_arn ← aws_iam_role.eks_node_role.arn      │
      │     └── subnet_ids ← module.vpc.subnet_ids                  │
      │                                                              │
      │   OUTPUTS: cluster_name, cluster_endpoint,                  │
      │             cluster_certificate_authority_data               │
      │                                                              │
      ├── module.ecr  (no dependencies — deploys in parallel)        │
      │     aws_ecr_repository.repos × 7 (for_each)                 │
      │     OUTPUTS: repository_urls map                             │
      │                                                              │
      │── data.aws_eks_cluster_auth.eks                              │
      │     ← module.eks.cluster_name                                │
      │     OUTPUT: token (short-lived JWT)                          │
      │                                                              │
      │── provider "kubernetes" alias="eks"                          │
      │     ← module.eks.cluster_endpoint                            │
      │     ← module.eks.cluster_certificate_authority_data          │
      │     ← data.aws_eks_cluster_auth.eks.token                    │
      │                                                              │
      │── provider "helm" alias="eks"                                │
      │     ← module.eks.cluster_endpoint                            │
      │     ← module.eks.cluster_certificate_authority_data          │
      │     ← data.aws_eks_cluster_auth.eks.token                    │
      │                                                              │
      └── module.argocd  (depends_on: module.eks)                   │
            USES: kubernetes.eks + helm.eks providers                │
            kubernetes_namespace_v1.argocd                           │
            kubernetes_namespace_v1.monitoring                       │
            helm_release.argocd ← kubernetes_namespace_v1.argocd    │
            helm_release.monitoring ← kubernetes_namespace_v1.monitoring
```

---

## Root Configuration

### `provider.tf`

**What it does:** Declares the only provider needed at the root level and enforces a minimum Terraform CLI version.

| Setting | Value | Why |
|---|---|---|
| `required_version` | `>= 1.5` | Terraform 1.5 introduced `check` blocks and improved moved/import blocks. Prevents running on older CLIs that lack these features. |
| `provider "aws"` | `hashicorp/aws ~> 5.0` | Version 5.x of the AWS provider introduced breaking changes in IAM/EKS APIs — the `~>` constraint allows patch updates (5.1, 5.2…) but not a jump to 6.x. |
| `region` | `var.region` (= `us-east-1`) | Parameterized so the entire stack can be redeployed to another region just by changing the tfvar. |

> **Note:** The `kubernetes` and `helm` providers are declared inside `main.tf` (not `provider.tf`) because they depend on runtime outputs from the `eks` module — their `host`, `token`, and `ca_certificate` values are not known until after the cluster is created.

---

### `variables.tf`

Declares the full interface for the root module. Every variable here maps to an entry in `terraform.tfvars`.

| Variable | Type | Purpose |
|---|---|---|
| `region` | `string` | AWS region for all resources |
| `vpc_name` | `string` | Name tag applied to VPC and derived resources |
| `vpc_cidr` | `string` | IP address range of the entire VPC (`10.1.0.0/16` = 65,534 hosts) |
| `subnets` | `list(object)` | Each subnet's name, CIDR block, and AZ. Object type enforces all three fields are always provided together. |
| `cluster_name` | `string` | Name of the EKS cluster; also used as a prefix in IAM role names |
| `node_group_name` | `string` | Name of the EKS managed node group |
| `instance_types` | `list(string)` | List lets AWS pick the best available instance matching the spec if one is unavailable |
| `capacity_type` | `string` | `ON_DEMAND` or `SPOT`. ON_DEMAND is used here for reliability; SPOT would be 60-90% cheaper but can be reclaimed. |
| `desired_size` | `number` | How many nodes to start with |
| `min_size` | `number` | Cluster Autoscaler will not scale below this |
| `max_size` | `number` | Cluster Autoscaler will not scale above this |
| `disk_size` | `number` | Root EBS volume size (GiB) per node |
| `repositories` | `list(string)` | Names of ECR repositories to create — one per microservice |

---

### `main.tf`

The orchestration layer. It wires all modules together and configures the dynamic providers.

**Module instantiation order enforced by Terraform:**

1. `module.vpc` and `module.ecr` — no inter-module dependencies, deployed in parallel
2. `module.eks` — `depends_on = [module.vpc]` ensures subnets exist before the cluster is created
3. `data.aws_eks_cluster_auth.eks` — reads a short-lived auth token after the cluster exists
4. `provider "kubernetes"` and `provider "helm"` — configured dynamically using EKS outputs
5. `module.argocd` — `depends_on = [module.eks]` ensures the cluster API is ready before Helm/K8s resources are applied

**Dynamic providers:**

```hcl
provider "kubernetes" {
  alias                  = "eks"
  host                   = module.eks.cluster_endpoint         # HTTPS API server URL
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)  # TLS trust
  token                  = data.aws_eks_cluster_auth.eks.token # Short-lived JWT from STS
}
```

The `alias = "eks"` means these providers are not the default — they are explicitly passed into `module.argocd` via `providers = { kubernetes = kubernetes.eks, helm = helm.eks }`. This pattern allows future modules to target different clusters without naming conflicts.

---

### `outputs.tf`

```
cluster_name      ← module.eks.cluster_name
cluster_endpoint  ← module.eks.cluster_endpoint
ecr_urls          ← module.ecr.repository_urls (map of name → URL)
```

These are published so CI/CD pipelines and other Terraform root modules can reference them via `terraform output` without reading module internals.

---

## Module: VPC

**Location:** `modules/vpc/`  
**Purpose:** Create the entire network foundation — every AWS resource that lives inside the VPC depends on this module existing first.

### Resources

#### 1. `aws_vpc.vpc`

```hcl
cidr_block           = "10.1.0.0/16"
enable_dns_support   = true
enable_dns_hostnames = true
```

**Why it exists:** A VPC is the mandatory network boundary for all AWS resources in this stack. Nothing can be deployed without it.

**Why DNS is enabled:** EKS requires `enable_dns_hostnames = true` so that worker node EC2 instances receive resolvable DNS names (e.g., `ec2-10-1-1-10.compute-1.amazonaws.com`). The EKS API server uses these hostnames internally to register nodes. Without this, the node group will fail to join the cluster.

**CIDR `10.1.0.0/16`:** Gives 65,534 available IP addresses. Using a `/16` leaves room to add many more subnets in the future. The `10.x.x.x` private range is the RFC 1918 standard.

---

#### 2. `aws_internet_gateway.igw`

```hcl
vpc_id = aws_vpc.vpc.id
```

**Why it exists:** Subnets in a VPC are isolated from the internet by default. The Internet Gateway is the single point of entry/exit for public traffic. Without it:
- Worker nodes cannot pull container images from ECR (which uses public HTTPS endpoints)
- ArgoCD cannot fetch manifests from GitHub
- The EKS control plane (managed by AWS) cannot communicate back to worker nodes over the public endpoint

Only one IGW can be attached to a VPC at a time.

**Connection:** Referenced by `aws_route_table.public` as the next-hop for all outbound traffic (`0.0.0.0/0`).

---

#### 3. `aws_subnet.subnets` (count = 3)

| Index | Name | CIDR | AZ |
|---|---|---|---|
| 0 | subnet-1 | `10.1.1.0/24` | `us-east-1a` |
| 1 | subnet-2 | `10.1.2.0/24` | `us-east-1b` |
| 2 | subnet-3 | `10.1.3.0/24` | `us-east-1c` |

```hcl
map_public_ip_on_launch = true
```

**Why 3 subnets across 3 AZs:** EKS requires subnets in at least 2 AZs for the control plane to operate in a highly available configuration. Using 3 AZs means if one AZ goes down, EKS can still schedule pods in the remaining two.

**Why `map_public_ip_on_launch = true`:** Worker nodes need a public IP to reach the internet (for image pulls, etc.) without a NAT Gateway. In a production environment with private subnets, you'd use a NAT Gateway instead — but this stack intentionally avoids NAT Gateway cost by using public subnets.

**Kubernetes-specific tags (critical):**

```
"kubernetes.io/role/elb" = "1"
"kubernetes.io/cluster/eks-cluster" = "owned"
```

- `kubernetes.io/role/elb = 1` — tells the AWS Load Balancer Controller that this subnet can host internet-facing Application Load Balancers (ALBs). Without this tag, `Ingress` objects in Kubernetes will fail to provision an ALB.
- `kubernetes.io/cluster/<name> = owned` — tells EKS that this subnet belongs to this specific cluster. AWS uses this to identify which subnets to use when provisioning load balancers and other cluster-managed resources.

---

#### 4. `aws_route_table.public`

```hcl
route {
  cidr_block = "0.0.0.0/0"
  gateway_id = aws_internet_gateway.igw.id
}
```

**Why it exists:** A route table is the routing rulebook for subnets. This one has a single rule: send all non-local traffic (`0.0.0.0/0`) out through the Internet Gateway. Without this, instances in the subnets have no route to the internet.

The VPC automatically creates a local route for `10.1.0.0/16` (intra-VPC traffic) — it does not need to be declared explicitly.

---

#### 5. `aws_route_table_association.public` (count = 3)

```hcl
subnet_id      = aws_subnet.subnets[count.index].id
route_table_id = aws_route_table.public.id
```

**Why it exists:** A route table has no effect until it is associated with a subnet. Without these associations, each subnet uses the VPC's default route table (which has no IGW route), and all internet traffic is silently dropped.

One association resource is created per subnet (count = 3).

---

### VPC Module Outputs

| Output | Value | Consumed by |
|---|---|---|
| `vpc_id` | ID of `aws_vpc.vpc` | Not used outside the module currently (reserved for future use) |
| `subnet_ids` | List of all 3 subnet IDs | `module.eks` → EKS cluster + node group placement |

---

## Module: EKS

**Location:** `modules/eks/`  
**Purpose:** Create the Kubernetes control plane, the worker nodes, the IAM permissions those components need, the OIDC identity provider (for pod-level IAM), and the EBS storage driver.

This is the most complex module — it provisions 13 resources with careful dependency ordering.

### Resources

#### 1. `aws_iam_role.eks_cluster_role`

```hcl
name = "eks-cluster-cluster-role"
Principal { Service = "eks.amazonaws.com" }
Action = "sts:AssumeRole"
```

**Why it exists:** The EKS control plane (managed by AWS) must act on your behalf to call AWS APIs — for example, to create Elastic Network Interfaces in your VPC subnets, or to describe EC2 instances when nodes register. This IAM role grants the EKS service that permission via a trust policy (`AssumeRole`).

Think of it as: "I allow the EKS service to wear this role's hat and call AWS APIs using my account's permissions."

---

#### 2. `aws_iam_role_policy_attachment.cluster_policy`

```hcl
role       = aws_iam_role.eks_cluster_role.name
policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
```

**Why it exists:** Attaches the AWS-managed `AmazonEKSClusterPolicy` to the cluster role. This policy grants EKS the specific permissions it needs: creating/deleting load balancers, describing EC2 instances and subnets, managing ENIs, updating auto-scaling groups, and writing to CloudWatch Logs.

Without this attachment, `aws_eks_cluster.eks` creation will fail with an IAM authorization error.

**Connection:** `aws_eks_cluster.eks` has `depends_on = [aws_iam_role_policy_attachment.cluster_policy]` — the cluster waits for this attachment to complete before being created.

---

#### 3. `aws_eks_cluster.eks`

```hcl
name     = "eks-cluster"
role_arn = aws_iam_role.eks_cluster_role.arn
version  = "1.34"

vpc_config {
  subnet_ids              = var.subnet_ids  # 3 subnets from VPC module
  endpoint_public_access  = true
  endpoint_private_access = false
}
```

**Why it exists:** This is the Kubernetes control plane itself — the API server, etcd, scheduler, and controller manager — all managed and operated by AWS. You never SSH into the control plane; you only interact with it via the `kubectl` API.

**Why `endpoint_public_access = true`:** The Kubernetes API server is accessible over the public internet (filtered by IAM auth). This is the default configuration that allows developers and CI/CD systems to run `kubectl` commands from outside the VPC. In a hardened setup, you'd enable `endpoint_private_access = true` and disable public access, requiring a VPN or bastion host.

**Version `1.34`:** Kubernetes 1.34 is a specific release pinned here. EKS supports N-2 minor versions; pinning prevents unexpected auto-upgrades.

**Outputs used by other resources:**
- `.endpoint` → kubernetes/helm providers, root `outputs.tf`
- `.certificate_authority[0].data` → kubernetes/helm providers, root `outputs.tf`
- `.identity[0].oidc[0].issuer` → OIDC provider + EBS CSI trust policy
- `.name` → node group, EBS CSI addon, `data.aws_eks_cluster_auth`

---

#### 4. `data.tls_certificate.eks`

```hcl
url = aws_eks_cluster.eks.identity[0].oidc[0].issuer
```

**Why it exists:** This data source fetches the TLS certificate from the EKS cluster's OIDC issuer endpoint and extracts the SHA-1 thumbprint. The thumbprint is required by AWS when registering an OIDC provider — it's how AWS verifies that the OIDC issuer URL actually belongs to your EKS cluster and hasn't been spoofed.

---

#### 5. `aws_iam_openid_connect_provider.eks`

```hcl
client_id_list  = ["sts.amazonaws.com"]
thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
url             = aws_eks_cluster.eks.identity[0].oidc[0].issuer
```

**Why it exists:** This is the foundation for **IRSA (IAM Roles for Service Accounts)** — the mechanism that lets individual Kubernetes pods assume specific IAM roles without any static AWS credentials.

How it works:
1. Each EKS cluster has a built-in OIDC issuer (a URL like `https://oidc.eks.us-east-1.amazonaws.com/id/XXXX`).
2. Registering this OIDC provider with AWS IAM tells AWS: "Trust JWTs issued by this Kubernetes cluster."
3. When a pod with a Kubernetes service account annotated with an IAM role ARN starts up, it gets a projected volume token (a short-lived JWT signed by the OIDC issuer).
4. The pod can exchange this JWT for temporary AWS credentials by calling `sts:AssumeRoleWithWebIdentity`.

This is used immediately for the EBS CSI driver IRSA role (resource #11 below), and can be used for any future pod that needs AWS access (e.g., pods that need to read from S3, write to DynamoDB, etc.).

**Connection:** Its ARN (`aws_iam_openid_connect_provider.eks.arn`) is used as the federated principal in the EBS CSI trust policy.

---

#### 6. `aws_iam_role.eks_node_role`

```hcl
name = "eks-cluster-node-role"
Principal { Service = "ec2.amazonaws.com" }
Action = "sts:AssumeRole"
```

**Why it exists:** Worker nodes are EC2 instances. Before they can join the cluster and pull workloads, they need permission to call AWS APIs: registering with the EKS cluster, configuring networking via the VPC CNI, and pulling container images from ECR. This role is assumed by EC2 at instance launch time (via the instance profile).

This is a different role from the cluster role — the cluster role is for the AWS-managed control plane, the node role is for your EC2 instances.

---

#### 7. `aws_iam_role_policy_attachment.worker_node_policy`

```hcl
policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
```

**Why it exists:** Grants the node role the core permissions for a worker node to operate within EKS:
- `eks:DescribeCluster` — get cluster info during bootstrap
- `ec2:Describe*` — describe network interfaces and instances
- Various permissions to register the node with the EKS control plane

Without this, nodes bootstrap but fail to register with the cluster.

---

#### 8. `aws_iam_role_policy_attachment.cni_policy`

```hcl
policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
```

**Why it exists:** Grants the node role permission to manage Elastic Network Interfaces (ENIs) and IP addresses on EC2 instances. This is required by the **AWS VPC CNI plugin** (`aws-node` DaemonSet), which runs on every node and is responsible for assigning pod IPs directly from the VPC subnet address space.

Key permissions granted:
- `ec2:AssignPrivateIpAddresses` — allocate pod IPs
- `ec2:AttachNetworkInterface` — attach secondary ENIs
- `ec2:CreateNetworkInterface` — create new ENIs for IP expansion
- `ec2:UnassignPrivateIpAddresses` — release pod IPs when pods terminate

Without this, pods cannot get IP addresses and will stay in `Pending` state.

---

#### 9. `aws_iam_role_policy_attachment.ecr_policy`

```hcl
policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
```

**Why it exists:** Grants the node role read-only access to all ECR repositories in the account. When Kubernetes schedules a pod with an ECR image URI, the kubelet on the node calls ECR's `GetAuthorizationToken` and `BatchGetImage` APIs to pull the image. Without this, image pulls fail with an authentication error.

This grants access to **all** ECR repos — in a stricter setup you could create a custom policy scoped to specific repository ARNs.

---

#### 10. `aws_eks_node_group.node_group`

```hcl
cluster_name    = aws_eks_cluster.eks.name      # "eks-cluster"
node_group_name = "eks-node-group"
node_role_arn   = aws_iam_role.eks_node_role.arn
subnet_ids      = var.subnet_ids                # 3 subnets from VPC module

instance_types = ["m7i-flex.large"]
capacity_type  = "ON_DEMAND"
disk_size      = 30

scaling_config {
  desired_size = 1
  min_size     = 1
  max_size     = 2
}

update_config {
  max_unavailable = 1
}
```

**Why it exists:** The managed node group is a group of EC2 instances that form the Kubernetes data plane — where your actual pods run. AWS manages node lifecycle (launch, health checks, replacement) using an Auto Scaling Group under the hood.

**Why `m7i-flex.large`:** The `m7i-flex` is Intel 7th-gen general-purpose, `flex` means variable CPU performance (slightly cheaper than full `m7i`). `large` = 2 vCPU, 8 GiB RAM — sufficient for running several small microservice containers.

**Why ON_DEMAND:** Spot instances can be reclaimed by AWS with 2 minutes notice. For a demo/dev cluster this could cause disruption. ON_DEMAND nodes run until you terminate them.

**Why `max_unavailable = 1`:** During a rolling update (e.g., upgrading the AMI), at most 1 node is replaced at a time. With `desired_size = 1`, this means the cluster will briefly have 0 nodes during an update — for production with `desired_size >= 2` this is a safe rolling strategy.

**`desired_size = min_size = 1, max_size = 2`:** Starts with 1 node. Cluster Autoscaler (if installed) can scale to 2 when pods are pending due to insufficient resources.

**`disk_size = 30` GiB:** Each worker node gets a 30 GiB root EBS volume. Container images and ephemeral storage consume this space. 30 GiB is adequate for a demo workload with several microservice images.

**Dependencies (must all complete before node group is created):**
- `aws_iam_role_policy_attachment.worker_node_policy`
- `aws_iam_role_policy_attachment.cni_policy`
- `aws_iam_role_policy_attachment.ecr_policy`

This ordering prevents a race condition where nodes launch before their role has the required permissions attached, causing bootstrap failures.

---

#### 11. `data.aws_iam_policy_document.ebs_csi_assume_role`

```hcl
actions = ["sts:AssumeRoleWithWebIdentity"]

principals {
  type        = "Federated"
  identifiers = [aws_iam_openid_connect_provider.eks.arn]
}

condition {
  test     = "StringEquals"
  variable = "<oidc-issuer-host>:sub"
  values   = ["system:serviceaccount:kube-system:ebs-csi-controller-sa"]
}
```

**Why it exists:** This generates the IAM trust policy document for the EBS CSI driver's IRSA role. The trust policy does two things:

1. **Allows OIDC federation:** Only JWTs issued by this cluster's OIDC provider can assume the role.
2. **Scopes to one service account:** The `StringEquals` condition on `:sub` means only the specific Kubernetes service account `ebs-csi-controller-sa` in namespace `kube-system` can assume this role — not any other pod on the cluster.

This principle of least privilege means even if a pod on the cluster is compromised, it cannot steal EBS permissions (unless that specific service account is compromised).

---

#### 12. `aws_iam_role.ebs_csi_irsa`

```hcl
name               = "eks-cluster-ebs-csi-irsa"
assume_role_policy = data.aws_iam_policy_document.ebs_csi_assume_role.json
```

**Why it exists:** This is the IAM role that the EBS CSI controller pod will assume at runtime via IRSA. It combines the trust policy (who can assume it) with the permissions policy (what they can do once they do).

---

#### 13. `aws_iam_role_policy_attachment.ebs_csi_irsa_policy`

```hcl
policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
```

**Why it exists:** Attaches the AWS-managed EBS CSI policy to the IRSA role. This policy grants the EBS CSI controller the specific permissions needed to manage EBS volumes on behalf of Kubernetes:
- `ec2:CreateVolume` — provision new EBS volumes when a `PersistentVolumeClaim` is created
- `ec2:AttachVolume` / `ec2:DetachVolume` — mount/unmount volumes to worker nodes
- `ec2:DeleteVolume` — delete volumes when PVCs are deleted
- `ec2:DescribeVolumes` — check volume state
- `ec2:CreateSnapshot` / `ec2:DeleteSnapshot` — for volume snapshots (VolumeSnapshotContent resources)
- `ec2:CreateTags` — tag created volumes for tracking

---

#### 14. `aws_eks_addon.ebs_csi`

```hcl
cluster_name             = aws_eks_cluster.eks.name
addon_name               = "aws-ebs-csi-driver"
service_account_role_arn = aws_iam_role.ebs_csi_irsa.arn
```

**Why it exists:** The EBS CSI (Container Storage Interface) driver is the bridge between Kubernetes `PersistentVolume` / `PersistentVolumeClaim` resources and actual EBS volumes in AWS. Without it:
- Kubernetes cannot dynamically provision EBS volumes for stateful workloads (databases, queues)
- `StorageClass` objects cannot create EBS-backed `PersistentVolumes`
- Existing volumes cannot be mounted/unmounted as pods move between nodes

Installing it as an **EKS Addon** (vs. a Helm chart) means AWS manages upgrades and ensures compatibility with the cluster's Kubernetes version.

The `service_account_role_arn` annotation wires the IRSA role to the addon's service account — this is what gives the CSI driver pod its AWS permissions without static credentials.

---

### EKS Module Outputs

| Output | Value | Consumed by |
|---|---|---|
| `cluster_name` | `aws_eks_cluster.eks.name` | `data.aws_eks_cluster_auth`, root `outputs.tf` |
| `cluster_endpoint` | `aws_eks_cluster.eks.endpoint` | `provider "kubernetes"`, `provider "helm"`, root `outputs.tf` |
| `cluster_certificate_authority_data` | `aws_eks_cluster.eks.certificate_authority[0].data` (base64) | `provider "kubernetes"`, `provider "helm"` |
| `node_group_name` | `aws_eks_node_group.node_group.node_group_name` | Informational |
| `node_group_arn` | `aws_eks_node_group.node_group.arn` | Informational |
| `node_group_status` | `aws_eks_node_group.node_group.status` | Informational |

---

## Module: ECR

**Location:** `modules/ecr/`  
**Purpose:** Create one AWS Elastic Container Registry repository per microservice. ECR is AWS's fully managed Docker image registry.

### Resource: `aws_ecr_repository.repos` (for_each = 7)

The `for_each = toset(var.repositories)` creates one independent repository resource for each name in the list:

| Repository Name | Purpose |
|---|---|
| `frontend` | React/web frontend application |
| `gateway` | API gateway / reverse proxy (e.g., Nginx, Kong, or custom) |
| `auth` | Authentication/authorization service |
| `order-service` | Service handling order creation and management |
| `orders` | Possibly a separate read model or order data service |
| `product-service` | Product catalog and inventory management |
| `user-service` | User profile management |

**Configuration details:**

```hcl
force_delete            = true   # Allows deleting the repo even if it contains images
scan_on_push            = true   # Automatic vulnerability scanning on every image push
image_tag_mutability    = "MUTABLE"  # Tags like "latest" can be overwritten
```

**Why `force_delete = true`:** In a dev/demo environment, `terraform destroy` should cleanly remove everything. By default, ECR refuses to delete a repo that contains images. `force_delete = true` overrides this and deletes contained images. In production you would set this to `false` to prevent accidental data loss.

**Why `scan_on_push = true`:** Amazon Inspector automatically scans each pushed image for known CVEs (Common Vulnerabilities and Exposures) using the OS package database and language package manifests. Results appear in the ECR console. This is a zero-effort security baseline.

**Why `MUTABLE` tags:** Allows CI/CD pipelines to push a new image with the same tag (e.g., `latest` or `main`) on every build. IMMUTABLE would require a unique tag per push (e.g., a git SHA), which is more secure but requires pipeline changes. MUTABLE is acceptable for development.

**Why ECR instead of Docker Hub:** ECR is natively integrated with EKS via the `AmazonEC2ContainerRegistryReadOnly` policy on the node role — no separate image pull secret is needed in Kubernetes. Images stay within the AWS network boundary.

---

### ECR Module Output

```hcl
output "repository_urls" {
  value = {
    for repo in aws_ecr_repository.repos :
    repo.name => repo.repository_url
  }
}
```

Produces a map like:
```json
{
  "frontend":        "123456789012.dkr.ecr.us-east-1.amazonaws.com/frontend",
  "gateway":         "123456789012.dkr.ecr.us-east-1.amazonaws.com/gateway",
  "auth":            "123456789012.dkr.ecr.us-east-1.amazonaws.com/auth",
  "order-service":   "123456789012.dkr.ecr.us-east-1.amazonaws.com/order-service",
  "orders":          "123456789012.dkr.ecr.us-east-1.amazonaws.com/orders",
  "product-service": "123456789012.dkr.ecr.us-east-1.amazonaws.com/product-service",
  "user-service":    "123456789012.dkr.ecr.us-east-1.amazonaws.com/user-service"
}
```

These URLs are used in CI/CD pipelines as the `docker push` target and in Kubernetes pod specs as the `image:` field.

---

## Module: ArgoCD

**Location:** `modules/argocd/`  
**Purpose:** Install ArgoCD (GitOps controller) and the kube-prometheus-stack (Prometheus + Grafana + Alertmanager) into the EKS cluster using Helm.

This module uses both the `kubernetes` and `helm` providers, which are passed in from the root module as aliased providers. It does not need any input variables — all configuration is hardcoded or defaults.

### Resources

#### 1. `kubernetes_namespace_v1.argocd`

```hcl
metadata { name = "argocd" }
```

**Why it exists:** Kubernetes namespaces provide logical isolation between workloads. The `argocd` namespace contains all ArgoCD components: the API server, repo server, application controller, and Redis. Creating the namespace as a Terraform resource (rather than letting the Helm chart create it with `create_namespace = true`) means the namespace lifecycle is managed by Terraform — it can be tracked, tagged, and deleted cleanly.

---

#### 2. `kubernetes_namespace_v1.monitoring`

```hcl
metadata { name = "monitoring" }
```

**Why it exists:** Same reasoning as the `argocd` namespace — provides isolation for Prometheus, Grafana, and Alertmanager components. The `helm_release.monitoring` resource explicitly depends on this namespace existing first.

---

#### 3. `helm_release.argocd`

```hcl
name       = "argocd"
namespace  = "argocd"
repository = "https://argoproj.github.io/argo-helm"
chart      = "argo-cd"
version    = "6.7.0"

values = [{
  server.service.type         = "ClusterIP"
  configs.params.server.insecure = true
}]
```

**Why ArgoCD:** ArgoCD is a GitOps continuous delivery tool for Kubernetes. It watches a Git repository for Kubernetes manifest changes (YAML/Helm) and automatically applies them to the cluster, keeping the actual cluster state in sync with the desired state declared in Git. This replaces manual `kubectl apply` steps in CI/CD pipelines.

**Why chart version `6.7.0`:** Version is pinned to ensure reproducible deployments. An unpinned `version = ""` would use the latest at plan time, which could break on re-apply if a breaking chart version was released.

**Why `service.type = "ClusterIP"`:** ArgoCD's web UI and gRPC API are exposed as a ClusterIP service (not a LoadBalancer or NodePort). This means the UI is only accessible from within the cluster or via `kubectl port-forward`. To expose it externally you'd add an Ingress resource later. ClusterIP is the most secure default.

**Why `server.insecure = true`:** By default ArgoCD serves its UI over HTTPS with a self-signed certificate. Setting `server.insecure = true` makes it serve over plain HTTP, which is common when you have a TLS-terminating Ingress controller (like Nginx or AWS ALB) in front of it — the outer layer handles HTTPS. Without this flag, the Ingress to ArgoCD requires special passthrough configuration.

---

#### 4. `helm_release.monitoring`

```hcl
name       = "kube-prometheus-stack"
namespace  = "monitoring"
repository = "https://prometheus-community.github.io/helm-charts"
chart      = "kube-prometheus-stack"
version    = "56.21.0"
timeout    = 600

values = [{
  grafana.service.type       = "ClusterIP"
  prometheus.service.type    = "ClusterIP"
  alertmanager.service.type  = "ClusterIP"
}]
```

**Why kube-prometheus-stack:** This is the community standard Kubernetes observability stack. A single Helm chart installs:
- **Prometheus:** Metrics collection engine. Scrapes metrics from all nodes, pods, and Kubernetes control plane components every 15 seconds.
- **Grafana:** Visualization dashboards. Comes pre-configured with 20+ dashboards showing node CPU/memory, pod health, namespace resource usage, etc.
- **Alertmanager:** Routes Prometheus alerts to notification channels (email, Slack, PagerDuty). Handles deduplication and grouping.
- **node-exporter DaemonSet:** Runs on every node to collect OS-level metrics (CPU, disk, network).
- **kube-state-metrics Deployment:** Exposes Kubernetes object state as metrics (pod status, deployment replica counts, etc.).

**Why `timeout = 600` (10 minutes):** The kube-prometheus-stack chart installs many CRDs (Custom Resource Definitions) and waits for them to become healthy before returning. Grafana also runs migrations on startup. 600 seconds provides enough headroom. The default 300-second timeout causes flaky failures on slower clusters.

**Why all services are `ClusterIP`:** Same rationale as ArgoCD — internal-only access, secured behind a future Ingress. LoadBalancer services would create one AWS NLB per service (expensive and unnecessary for this stack).

**`depends_on = [kubernetes_namespace_v1.monitoring]`:** Explicitly waits for the namespace to exist before the Helm release tries to deploy into it, preventing a race condition.

---

## Data Flow Between Modules

This section traces exactly which value flows from where to where, crossing module boundaries.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     terraform.tfvars                                 │
│  vpc_name, vpc_cidr, subnets[] → module.vpc                         │
│  cluster_name, node_group_name, instance_types,                      │
│  min/desired/max_size, disk_size → module.eks                        │
│  repositories[] → module.ecr                                         │
│  (no inputs to module.argocd — it reads from providers)              │
└─────────────────────────────────────────────────────────────────────┘

module.vpc.subnet_ids  ──────────────────────────────────────────────►  module.eks
                                                                         ├── aws_eks_cluster.eks.vpc_config.subnet_ids
                                                                         └── aws_eks_node_group.node_group.subnet_ids

module.eks.cluster_name  ───────────────────────────────────────────►  data.aws_eks_cluster_auth.eks.name
                                                                         └── (gets a short-lived auth token)

module.eks.cluster_endpoint  ───────────────────────────────────────►  provider "kubernetes" { host }
                                                                         provider "helm"       { host }
                                                                         outputs.tf            { cluster_endpoint }

module.eks.cluster_certificate_authority_data  ─────────────────────►  provider "kubernetes" { cluster_ca_certificate }
                                                                         provider "helm"       { cluster_ca_certificate }

data.aws_eks_cluster_auth.eks.token  ───────────────────────────────►  provider "kubernetes" { token }
                                                                         provider "helm"       { token }

provider "kubernetes" (alias=eks)  ─────────────────────────────────►  module.argocd (kubernetes_namespace resources)
provider "helm"       (alias=eks)  ─────────────────────────────────►  module.argocd (helm_release resources)

module.ecr.repository_urls  ────────────────────────────────────────►  outputs.tf { ecr_urls }
                                                                         (consumed by CI/CD pipelines externally)
```

---

## Actual Values (terraform.tfvars)

| Variable | Value | Notes |
|---|---|---|
| `region` | `us-east-1` | AWS Northern Virginia |
| `vpc_name` | `EKS-Demo-VPC` | Name tag on VPC and related resources |
| `vpc_cidr` | `10.1.0.0/16` | 65,534 usable IP addresses |
| `subnets[0]` | `10.1.1.0/24` / `us-east-1a` | 254 IPs in AZ-a |
| `subnets[1]` | `10.1.2.0/24` / `us-east-1b` | 254 IPs in AZ-b |
| `subnets[2]` | `10.1.3.0/24` / `us-east-1c` | 254 IPs in AZ-c |
| `cluster_name` | `eks-cluster` | Also used as IAM role name prefix |
| `node_group_name` | `eks-node-group` | ASG name in EC2 console |
| `instance_types` | `["m7i-flex.large"]` | 2 vCPU, 8 GiB RAM |
| `capacity_type` | `ON_DEMAND` | No spot interruption risk |
| `desired_size` | `1` | Start with 1 node |
| `min_size` | `1` | Never scale below 1 |
| `max_size` | `2` | Can scale to 2 nodes |
| `disk_size` | `30` GiB | Root EBS volume per node |
| `repositories` | 7 names | frontend, gateway, auth, order-service, orders, product-service, user-service |

---

## Outputs Exposed to the Caller

These are the values available via `terraform output` after a successful apply:

| Output | Source | Example Value | Use Case |
|---|---|---|---|
| `cluster_name` | `module.eks.cluster_name` | `eks-cluster` | `aws eks update-kubeconfig --name eks-cluster` |
| `cluster_endpoint` | `module.eks.cluster_endpoint` | `https://XXXX.gr7.us-east-1.eks.amazonaws.com` | kubectl, CI/CD pipeline API calls |
| `ecr_urls` | `module.ecr.repository_urls` | Map of 7 repo URLs | CI/CD `docker push` targets, K8s pod `image:` specs |

---

## Dependency Execution Order Summary

When you run `terraform apply`, resources are created in this approximate order:

```
Phase 1 (parallel):
  ├── module.vpc: aws_vpc
  └── module.ecr: aws_ecr_repository × 7

Phase 2 (depends on vpc):
  └── module.vpc: aws_internet_gateway, aws_subnet × 3

Phase 3 (depends on subnets):
  └── module.vpc: aws_route_table, aws_route_table_association × 3

Phase 4 (depends on module.vpc):
  └── module.eks: aws_iam_role.eks_cluster_role + aws_iam_role.eks_node_role

Phase 5 (depends on IAM roles):
  └── module.eks: aws_iam_role_policy_attachment × 4 (cluster + 3 node policies)

Phase 6 (depends on cluster_policy attachment):
  └── module.eks: aws_eks_cluster.eks
  
Phase 7 (depends on EKS cluster):
  └── module.eks: data.tls_certificate.eks

Phase 8 (depends on TLS cert):
  └── module.eks: aws_iam_openid_connect_provider.eks

Phase 9 (parallel, depends on OIDC provider + all node policy attachments):
  ├── module.eks: aws_eks_node_group.node_group
  └── module.eks: data.aws_iam_policy_document.ebs_csi_assume_role

Phase 10 (depends on ebs_csi assume role doc):
  └── module.eks: aws_iam_role.ebs_csi_irsa

Phase 11 (depends on ebs_csi role):
  └── module.eks: aws_iam_role_policy_attachment.ebs_csi_irsa_policy

Phase 12 (depends on node group + ebs_csi policy):
  └── module.eks: aws_eks_addon.ebs_csi

Phase 13 (depends on module.eks complete):
  └── data.aws_eks_cluster_auth.eks → provider "kubernetes" + provider "helm"

Phase 14 (depends on providers being configured):
  └── module.argocd: kubernetes_namespace_v1.argocd + kubernetes_namespace_v1.monitoring

Phase 15 (depends on namespaces):
  └── module.argocd: helm_release.argocd + helm_release.monitoring
```

Total approximate resources: **28 AWS/K8s/Helm resources** across 4 modules.
