# EKS Cluster Archetype (CloudFormation)

CloudFormation template ("arquetipo") for a private Amazon EKS cluster in an existing VPC, with a managed
node group, a Fargate profile, managed add-ons and an SSM-managed bastion whose user data installs the
AWS Load Balancer Controller and deploys demo apps behind public and internal ALBs.

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

- **EKS cluster** (default 1.33): private endpoint only, auth mode `API_AND_CONFIG_MAP`, `api` and
  `audit` logs, placed in the three compute (private) subnets.
- **Managed node group**: on-demand, Bottlerocket by default, fixed size (`NumWorkerNodes` = min = max).
- **Fargate profile** for the `fargate` namespace, with its pod execution role.
- **Managed add-ons** with pinned versions: vpc-cni, kube-proxy, coredns, aws-ebs-csi-driver,
  amazon-cloudwatch-observability, metrics-server, cert-manager, aws-privateca-connector-for-kubernetes,
  eks-node-monitoring-agent.
- **Security groups** for control plane, worker nodes and bastion, with explicit 443/10250 rules.
- **IAM roles**: control plane, worker nodes, Fargate pod execution, bastion (plus instance profile and an
  EKS access entry granting the bastion role `AmazonEKSClusterAdminPolicy`).
- **Bastion EC2** (`t3a.large`, compute subnet 1, tagged with the `pTag*` values). Its user data installs
  AWS CLI v2, kubectl, eksctl and Helm, then: creates the IAM OIDC provider, the Load Balancer Controller
  IAM policy and service account, installs the controller via Helm, tags public/internal subnets for ELB
  discovery, and deploys nginx (internet-facing ALB, `nginx.<PublicDomain>`) and httpd (internal ALB,
  `apache.<PrivateDomain>`) in the `fargate` namespace, each with an HPA.

## Parameters

| Name | Purpose | Default |
|------|---------|---------|
| `VPC` | Existing VPC ID | `<VPC_ID>` |
| `PublicSubnet1..3` | Public subnets, tagged for internet-facing ALBs | `<SUBNET_ID>` |
| `ComputeSubnet1..3` | Private subnets for cluster, nodes, Fargate and bastion | `<SUBNET_ID>` |
| `InternalSubnet1..3` | Subnets tagged for internal ALBs | `<SUBNET_ID>` |
| `PublicDomain` / `PublicCertificateARN` | Public domain and ACM cert for the public ingress | `<DOMAIN>` / `<CERTIFICATE_ARN>` |
| `PrivateDomain` / `PrivateCertificateARN` | Private domain and ACM cert for the internal ingress | `<DOMAIN>` / `<CERTIFICATE_ARN>` |
| `EKSClusterName` | Cluster name and resource name prefix | `cluster-eks-` |
| `EKSVersion` | Kubernetes version | `1.33` |
| `NumWorkerNodes` / `NodeInstanceType` / `DiskSize` | Node group size, type, disk GiB | `3` / `t3a.large` / `100` |
| `WorkersAmiType` | Node AMI type (e.g. `AL2_x86_64`) | `BOTTLEROCKET_x86_64` |
| `BastionAmiId` | Amazon Linux AMI for the bastion | `<AMI_ID>` |
| `pTagValue*` | Corporate tags on the bastion (owner emails, cost center, backup, criticity...) | `<EMAIL>` for owner tags |

## Deploy

Defaults are environment-specific: override every network, domain, certificate, AMI and email parameter.

```bash
aws cloudformation deploy \
  --stack-name eks-archetype \
  --template-file arquetipo-eks-cloudformation.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
      VPC=<VPC_ID> EKSClusterName=<CLUSTER_NAME> BastionAmiId=<AMI_ID> \
      PublicSubnet1=<SUBNET_ID> PublicSubnet2=<SUBNET_ID> PublicSubnet3=<SUBNET_ID> \
      ComputeSubnet1=<SUBNET_ID> ComputeSubnet2=<SUBNET_ID> ComputeSubnet3=<SUBNET_ID> \
      InternalSubnet1=<SUBNET_ID> InternalSubnet2=<SUBNET_ID> InternalSubnet3=<SUBNET_ID> \
      PublicDomain=<DOMAIN> PublicCertificateARN=<CERTIFICATE_ARN> \
      PrivateDomain=<DOMAIN> PrivateCertificateARN=<CERTIFICATE_ARN> \
      pTagValueTechOwner=<EMAIL> pTagValueBussOwner=<EMAIL>
```

Add-on versions are pinned for 1.33. The compute subnets need outbound internet (NAT) for the bastion
bootstrap; follow it in `/root/eks-user-data.log` via SSM.

## Security notes

- **Bastion IAM allows privilege escalation**: `iam:CreateRole`, `iam:CreatePolicy`, `iam:AttachRolePolicy`
  and `iam:PassRole` on `*` (plus `cloudformation:CreateStack`), and it is a cluster admin.
- Bastion security group allows TCP 443 from `0.0.0.0/0` (instance is in a private subnet).
- Worker role has `AmazonEC2ContainerRegistryPowerUser` (can push images to ECR).
- User data downloads unpinned tooling (latest kubectl/eksctl, Helm install script and the controller IAM
  policy from `main`); demo images use `:latest`, and the nginx demo is exposed through an internet-facing ALB.

## Cleanup

Resources created by the bastion user data are not managed by this stack. Delete them first:

```bash
kubectl delete ingress --all -n fargate   # removes the ALBs created by the controller
aws cloudformation delete-stack --stack-name eks-archetype
```

Then remove manually: the eksctl service-account stack (`eksctl-<CLUSTER_NAME>-addon-iamserviceaccount-...`),
the IAM policy `AWSLoadBalancerControllerIAMPolicy-<CLUSTER_NAME>`, the cluster's IAM OIDC provider, and
the `kubernetes.io/*` tags added to the subnets.

## License

MIT — see [LICENSE](LICENSE).
