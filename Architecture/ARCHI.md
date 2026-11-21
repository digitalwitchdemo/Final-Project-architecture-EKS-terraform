# Architecture: AWS EKS Platform

## Overview

This repository provisions an AWS-hosted Kubernetes platform using Terraform. Terraform is the control plane for infrastructure creation, while Kubernetes and Helm operate the workloads and platform components inside EKS.

The platform consists of five connected systems:

1. AWS network infrastructure
2. Amazon EKS and managed worker nodes
3. Terraform state and automation
4. Kubernetes platform services
5. Public DNS, ingress, and TLS delivery

This is primarily an infrastructure platform repository. It does not contain application source code or a complete application deployment repository.

---

## Fundamental Control Flow

```text
GitHub / GitHub Actions
          |
          | Terraform providers and OIDC role
          v
Amazon Web Services
          |
          +--> VPC, subnets, routes, security groups
          |
          +--> EKS control plane
          |         |
          |         +--> Managed EC2 worker nodes
          |
          +--> EFS, EBS, IAM, S3
          |
          v
Kubernetes API
          |
          +--> cert-manager
          +--> NGINX Ingress Controller
          +--> Argo CD
          +--> Storage add-ons
```

At the highest level:

- Terraform describes the desired AWS and Kubernetes state.
- AWS providers create and manage AWS resources.
- The Kubernetes and Helm providers operate resources inside EKS.
- Worker nodes execute Kubernetes workloads.
- Ingress exposes selected services through AWS Load Balancer infrastructure.
- Argo CD provides the operational and GitOps interface.

---

## 1. Terraform Resource Model

Terraform configuration is divided into numbered files so dependencies and responsibilities remain readable.

| File range | Responsibility |
|---|---|
| `00-provider.tf` | Providers, authentication, backend, and S3 state |
| `01-vpc.tf`–`08-keypair.tf` | AWS network, security, routing, and access foundations |
| `09-eks-cluster.tf` | EKS control plane and cluster access |
| `10-eks-node-group.tf` | EC2 worker nodes and node IAM |
| `11-outputs.tf` | Operational values exposed after deployment |
| `12-cert-manager.tf`–`15-argocd.tf` | Ingress, certificates, storage, and Argo CD |
| `16-kubeconfig.tf`–`17-ebs.tf` | Administrative access and EBS storage |

Terraform resolves dependencies from resource references rather than from filename order alone. Files are separated primarily for maintainability.

### Providers

`00-provider.tf` configures:

- AWS provider
- Kubernetes provider
- Helm provider
- Encrypted S3 remote backend

The Kubernetes and Helm providers authenticate using the EKS cluster endpoint, certificate authority data, and a short-lived EKS authentication token.

### Remote State

Terraform stores state in S3 under:

```text
s3://<bucket>/eks/terraform.tfstate
```

The backend enables encryption and S3-native locking. State contains sensitive deployment information and must not be committed to Git or shared publicly.

---

## 2. Network Architecture

### VPC

The configuration creates a dedicated VPC with:

- Two public subnets
- Two private subnets
- One public and one private network tier
- Resources distributed across two Availability Zones

The public subnets support internet-facing infrastructure. The private subnets host the EKS worker nodes and limit direct public access.

### Routing and Internet Access

```text
Public subnet
  |
  v
Internet gateway
  |
  v
Internet

Private subnet
  |
  v
NAT gateway
  |
  v
Internet gateway
```

Worker nodes use NAT gateways for outbound internet access. They do not receive public IPs through the private subnet design.

### Security Boundaries

The EKS API endpoint is both public and private. Public API access is restricted by configurable public CIDRs. Cluster authentication is separate from network reachability:

1. A caller must reach the endpoint from an allowed CIDR.
2. AWS and Kubernetes authorization must then recognize the caller.

Application security groups restrict service access using configurable ingress CIDRs.

The EFS security group permits NFS traffic, TCP port `2049`, only from inside the VPC CIDR.

---

## 3. EKS Control Plane

`09-eks-cluster.tf` creates the EKS control plane and associated IAM role.

The cluster:

- Uses the configured cluster name and Kubernetes version
- Runs across public and private subnets
- Enables public and private API endpoints
- Restricts public endpoint CIDRs through variables
- Uses AWS IAM plus the Kubernetes access configuration
- Automatically grants cluster-admin access to the cluster creator

### Cluster Authentication

The cluster uses `API_AND_CONFIG_MAP` authentication mode. Terraform creates EKS access entries and associates the Amazon EKS Cluster Admin policy with configured IAM principals.

This produces two authorization layers:

| Layer | Purpose |
|---|---|
| AWS EKS access entry | Maps an IAM principal to the cluster |
| Kubernetes RBAC | Defines what Kubernetes operations the principal can perform |

The cluster creator receives bootstrap administrator permissions automatically. Additional IAM users or roles can receive cluster-admin access through `cluster_admin_arns`.

### Responsibility Split

```text
AWS manages:
- EKS control-plane infrastructure
- EKS API availability
- EKS authentication integration
- Managed node lifecycle

Kubernetes manages:
- Pod scheduling
- Workloads and deployments
- Services and Ingress resources
- Storage claims and mounted volumes
```

---

## 4. Managed Worker Nodes

`10-eks-node-group.tf` creates an Amazon EKS managed node group.

Worker nodes:

- Run in private subnets
- Use the configured instance types
- Use SPOT or ON_DEMAND capacity
- Scale between configured minimum and maximum sizes
- Receive the required worker, CNI, and container registry permissions
- Use one worker node IAM role shared by EC2 instances

### Node IAM

The node role receives:

- Amazon EKS worker-node policy
- Amazon EKS CNI policy
- Amazon EC2 Container Registry read-only access

The CNI policy allows Kubernetes networking components to manage pod networking. The node role is distinct from the service IAM roles used by Kubernetes add-ons.

### Pod Placement

Kubernetes schedules pods onto worker nodes based on:

- Available CPU and memory
- Node health
- Scheduling constraints
- Taints and tolerations, if any
- Pod affinity or anti-affinity, if defined

Managed node-group scaling operates at the EC2 instance level. Kubernetes controls which workloads run on those instances.

---

## 5. Storage Architecture

The platform provides two storage types.

| Storage | Purpose | Scope |
|---|---|---|
| Amazon EBS CSI | Node or workload block storage | Typically tied to an Availability Zone |
| Amazon EFS CSI | Shared, dynamically provisioned file storage | Shared across supported nodes and zones |

### Amazon EBS

`17-ebs.tf` installs the AWS EBS CSI add-on. This enables Kubernetes workloads to request persistent block volumes backed by Amazon EBS.

EBS-backed volumes are appropriate for workloads that require block storage. They may have Availability Zone placement constraints that must be considered when pods move between nodes.

### Amazon EFS

`14-efs.tf` creates:

- An encrypted EFS file system
- One mount target in each private subnet
- An EFS-specific security group
- Pod Identity Agent support
- An IAM role for the EFS CSI controller
- A managed EFS CSI add-on
- The `efs-sc` StorageClass

### EFS Provisioning Flow

```text
Application requests PVC
          |
          v
Kubernetes PVC references efs-sc
          |
          v
EFS CSI controller receives provisioning request
          |
          v
EFS access point is created
          |
          v
PV is created and PVC becomes Bound
          |
          v
Pod mounts EFS through the worker node network
```

The StorageClass uses:

- The `efs.csi.aws.com` provisioner
- `efS-ap` provisioning mode
- A file system ID
- Directory permissions
- A generated dynamic provisioning base path
- Immediate volume binding
- Delete reclamation policy

### EFS IAM Boundary

Pod Identity Agent associates the EFS CSI controller service account with an AWS IAM role. That role receives the Amazon EFS CSI driver policy.

This separation allows the controller to call AWS APIs without placing AWS credentials inside application pods.

---

## 6. Public Request Flow

The public delivery path for Argo CD is:

```text
Browser
  |
  v
DNS record
  |
  v
AWS Network Load Balancer
  |
  v
NGINX Ingress Controller
  |
  v
Argo CD Ingress rule
  |
  v
Argo CD server Service
  |
  v
Argo CD server Pod
```

### DNS

A DNS CNAME maps the public hostname to the hostname assigned to the AWS Network Load Balancer. DNS does not itself route traffic into Kubernetes; it resolves the hostname to the load balancer endpoint.

### Load Balancer

F5 NGINX Ingress Controller creates or manages an AWS Network Load Balancer service. The load balancer receives public traffic and forwards it to ingress controller pods in the cluster.

### NGINX

NGINX evaluates Ingress resources and routes traffic based on:

- Hostname
- HTTP path
- Backend Service name
- Backend port
- TLS configuration

### TLS

cert-manager manages certificate requests and renewals.

The request sequence is:

```text
Ingress requests a certificate
          |
          v
cert-manager creates a CertificateRequest
          |
          v
Let's Encrypt ClusterIssuer receives an ACME request
          |
          v
HTTP/TLS ownership challenge completes
          |
          v
Certificate is issued and stored as a Kubernetes Secret
          |
          v
NGINX terminates HTTPS using the certificate
```

The repository is configured with a production Let’s Encrypt issuer and an optional contact email.

### Ingress Installation Dependency

Argo CD is intentionally dependent on the NGINX and cert-manager resources in Terraform. NGINX and certificates must exist before the public Argo CD route is useful.

The Argo CD Helm chart disables automatic Ingress creation. `manifests/argocd-ingress.yaml` creates the Ingress separately after DNS is available.

---

## 7. Argo CD Architecture

Argo CD is installed with Terraform and Helm into the `argocd` namespace.

The chart configuration:

- Sets the Argo CD domain
- Runs the Argo CD server in insecure mode inside the cluster
- Lets NGINX terminate TLS externally
- Adds custom application health checks
- Disables Helm-managed chart Ingress creation

Inside the cluster, the architecture generally consists of:

- Argo CD server
- Repository management components
- Application controller components
- Redis
- Supporting Services and Secrets

The initial administrator password is stored in `argocd-initial-admin-secret`. Terraform exposes the password as a sensitive output.

Argo CD is the only application-facing interface deployed directly by this repository. It does not receive application source code from this repository unless Argo CD is later configured to watch external repositories or manifests.

---

## 8. Kubernetes and AWS Identity Separation

The project uses several identities for different purposes.

| Identity | Purpose |
|---|---|
| Terraform/AWS identity | Creates AWS infrastructure and Kubernetes provider resources |
| EKS cluster creator | Automatically receives bootstrap cluster-admin access |
| Cluster administrator IAM principals | Receive Amazon EKS cluster-admin policy access |
| Worker node role | Allows EC2 nodes to join EKS and use managed AWS services |
| EFS CSI role | Allows the storage controller to manage EFS resources |
| GitHub Actions OIDC role | Authenticates CI/CD to AWS |

Separating identities limits each component to the operations it is intended to perform.

The Kubernetes provider is not an AWS user identity. It uses a short-lived EKS token to call the Kubernetes API.

---

## 9. CI/CD Operating Flow

GitHub Actions is the automation entry point for deployment and teardown.

### Normal Deployment

```text
Push to main or manual workflow run
          |
          v
Checkout repository
          |
          v
Set up Terraform
          |
          v
Assume AWS OIDC role
          |
          v
terraform init
          |
          v
terraform fmt -check
          |
          v
terraform validate
          |
          v
tfsec soft-fail scan
          |
          v
tfsec soft-fail scan
          |
          v
terraform plan -out=tfplan
          |
          v
terraform apply -auto-approve tfplan
```

A concurrency group prevents simultaneous Terraform runs against the same state.

The workflow authenticates without committed AWS access keys by using GitHub OIDC and an IAM role secret.

### Destroy Flow

A manual workflow run can set `destroy=true`.

```text
Manual dispatch with destroy=true
          |
          v
Assume AWS role
          |
          v
terraform init
          |
          v
terraform destroy -auto-approve
```

The plan-and-apply job is skipped when destruction is selected.

A post-deployment test job exists in commented form. It does not currently run and should not be treated as an active validation stage.

---

## 10. Kubeconfig and Administrative Access

`16-kubeconfig.tf` creates a kubeconfig containing:

- Cluster endpoint
- Cluster CA certificate
- AWS CLI-based authentication command
- Cluster name

The kubeconfig is uploaded to S3 under the configured key.

The generated file grants access according to the cluster identity and access entries configured for the caller. It is not a Kubernetes secret by itself, but it behaves like one because it can provide administrator-level access.

---

## 11. Terraform Dependency Chain

A simplified deployment dependency chain is:

```text
S3 backend
  |
  v
VPC + subnets + routing
  |
  v
Security groups + IAM roles
  |
  v
EKS control plane
  |
  v
Managed worker nodes
  |
  v
EKS add-ons and Kubernetes providers
  |
  +--> cert-manager
  +--> NGINX
  +--> EFS CSI
  +--> EBS CSI
  +--> Argo CD
  |
  v
DNS + Ingress + Let's Encrypt
  |
  v
Public Argo CD service
```

Terraform provider resources may depend on the EKS endpoint, and Kubernetes resources may depend on worker nodes and add-ons. This creates a circular-looking dependency chain at a high level, so provider resources must be ordered carefully and Helm releases must declare explicit dependencies.

---

## 12. Operational States

### Infrastructure Healthy

Indicators include:

- EKS cluster status is active
- Worker nodes are `Ready`
- Core system pods are running
- State can be read and written
- Node IAM roles are operational
- EFS mount targets are available
- Add-on rollouts complete successfully

### Public Service Healthy

Indicators include:

- DNS resolves to the load balancer
- Port 80 redirects or serves HTTP
- Port 443 presents a valid certificate
- NGINX routes the configured hostname
- cert-manager reports a ready Certificate
- Argo CD pods and Services are healthy

### Degraded by Boundary

Most failures can be classified at one boundary:

| Boundary | Example failure |
|---|---|
| AWS account or IAM | Credentials rejected or resource creation denied |
| Remote state | State unavailable or locking conflict |
| Network | Private nodes cannot reach NAT or EFS |
| EKS control plane | API unavailable or caller unauthorized |
| Kubernetes scheduling | Pods pending or nodes unhealthy |
| Storage | PVC remains pending or cannot mount |
| DNS | Hostname does not resolve to load balancer |
| TLS | Certificate request or challenge fails |
| Application | Argo CD pod or Service unhealthy |

This boundary model prevents infrastructure observations from being confused with application behavior.

---

## 13. Security Model

### Network Security

- Worker nodes run in private subnets
- EKS public API access is limited by CIDRs
- EFS accepts NFS traffic only from the VPC
- Public ingress is concentrated through NGINX

### Identity Security

- OIDC replaces static GitHub Actions AWS keys
- Worker instances use a node IAM role
- CSI controllers use separate IAM roles
- Cluster-admin access is explicit through access entries

### Secret and State Security

- Terraform state is encrypted in S3
- S3 public access is disabled
- Kubeconfigs are stored outside Git
- AWS credentials are not embedded in configuration
- Argo CD initial credentials are temporary and sensitive

### Remaining Security Concerns

- tfsec runs in soft-fail mode
- IAM permissions may be broader than individual workloads require
- The public EFS export permits VPC-wide NFS access by CIDR
- Environment values remain account-specific unless separated
- State separation between environments is not implemented
- Long-lived initial Argo CD credentials need removal or rotation

---

## 14. Scope and Architectural Limitations

The platform implements infrastructure and operational tooling, but it does not implement:

- Application source code
- A backend service
- A frontend application
- Application databases
- Application-specific CI/CD
- Runtime observability dashboards
- Centralized logging
- Backup and restore automation
- Secrets automation
- Cross-account environment isolation
- High-availability application workloads

A complete production architecture would add application delivery, observability, secrets management, environment separation, backup, and recovery controls around this EKS foundation.

---

## 15. Summary

The project operates as a layered system:

```text
Terraform
  creates
    AWS infrastructure
      hosts
        EKS Kubernetes platform
          runs
            NGINX, cert-manager, Argo CD, storage add-ons
              exposes
                HTTPS operational interface
```

The core distinction is that AWS provides infrastructure and identity, Kubernetes provides workload orchestration, Terraform describes desired state, and external DNS plus NGINX provide public access. These layers are interconnected but have separate failure domains and should be diagnosed independently.
