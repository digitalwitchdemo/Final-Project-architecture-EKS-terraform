# AWS EKS Platform with Terraform

We completed this project as a team to build and document a production-oriented Kubernetes platform on AWS. Terraform delivered the networking, EKS cluster, worker nodes, storage, ingress, TLS, Argo CD, and supporting access configuration for learning and practical DevOps use.

## What we built

We created a VPC with two public subnets and two private subnets across two Availability Zones. The internet gateway served public subnet traffic, while NAT gateways allowed the private worker nodes to reach external services.

We deployed an Amazon EKS cluster across the VPC, placed the managed worker-node group in the private subnets, and configured restricted cluster and application access. We also used AWS and Kubernetes IAM integration, the EBS CSI add-on, and Pod Identity for the EFS CSI controller.

For platform services, we:

- Installed F5 NGINX Ingress Controller behind an AWS Network Load Balancer.
- Installed cert-manager and a production Let's Encrypt ClusterIssuer.
- Deployed Argo CD and published it through `argocd.pod7devopsengineers.online`.
- Provisioned encrypted Amazon EFS with private mount targets and an `efs-sc` StorageClass.
- Generated an administrator kubeconfig and uploaded it to Amazon S3.
- Added a GitHub Actions workflow that initialized, formatted, validated, scanned, planned, and applied the Terraform configuration, as well as performed manual destruction.

## Architecture

```text
Internet
   |
AWS Network Load Balancer
   |
F5 NGINX Ingress Controller
   |
Kubernetes workloads / Argo CD
   |
Amazon EKS worker nodes (private subnets)
   |                         |
Amazon EFS               Amazon EBS
```

The platform followed a simple division of responsibilities:

- `00-provider.tf`–`08-keypair.tf` created the provider, backend, networking, security, and key-pair foundations.
- `09-eks-cluster.tf` and `10-eks-node-group.tf` created the Kubernetes control plane and private worker nodes.
- `12-cert-manager.tf`–`15-argocd.tf` installed ingress, TLS, storage, and deployment tooling.
- `16-kubeconfig.tf` generated cluster access and `17-ebs.tf` enabled AWS EBS storage.
- `charts/`, `manifests/`, `efs-test/`, and `jira/` held supporting Kubernetes, Helm, validation, and project assets.

## Automation

The `.github/workflows/terraform.yml` workflow ran Terraform through GitHub Actions on pushes to `main` and on manual runs. It authenticated with AWS using OIDC and a repository secret (`AWS_OIDC_ARN`).

The workflow used S3 as the remote state backend and S3-native state locking. It formatted and validated Terraform, ran tfsec as a soft-fail security scan, and applied the saved plan. A manual destroy input provided an explicit teardown path. The post-deployment test job remained prepared but commented out.

## Scope

This project covered infrastructure and platform foundations; it did not include application source code or a separate application deployment pipeline. Argo CD was deployed as an operating component, while applications could be added through Kubernetes or GitOps later.

The environment was account-specific. The checked-in defaults referenced AWS account `409415529627`, region `us-west-2`, the `terraformstate-bucket-409415529627-us-west-2-an` S3 bucket, and the `argocd.pod7devopsengineers.online` hostname. Those values needed to be changed when the platform was reused in another account or domain.

## How we verified the platform

We used Terraform validation and planning as the primary configuration checks. We also used Kubernetes checks for nodes, pods, the NGINX ingress, cert-manager, Argo CD, and the `efs-sc` StorageClass. The `efs-test/` directory contained the resources used to validate dynamic EFS provisioning.

Useful access and verification commands were:

```bash
aws eks update-kubeconfig --region us-west-2 --name pod7cloud-eks
kubectl get nodes
kubectl get pods -A
kubectl get storageclass
terraform output
terraform output -raw argocd_initial_admin_password
```

The initial Argo CD administrator credential was sensitive and handled through Terraform output. We changed or removed that initial credential as part of the operating process.

## Security and environment notes

The repository did not contain AWS credentials or private key material. We retained least-privilege AWS identities, restricted cluster and application access with configurable CIDRs, and kept the Terraform state and kubeconfig in protected S3 resources.

EFS encrypted data at rest and exposed NFS port `2049` only to traffic from the VPC. The cluster used public API access only within the configured public CIDR list, and private worker nodes obtained egress through NAT gateways.
