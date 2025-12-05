**Networking & Security Design — AWS Microservices Architecture**

Purpose
- Provide a secure, production-ready VPC and network topology, with controls for connectivity, segmentation, and secrets management.
- Cover VPC layout, subnets, routing, security groups, NACLs, endpoints, private connectivity, secrets, IAM patterns, and monitoring.

1) VPC & Subnet Topology
- Single VPC per-account or per-environment (`staging`, `prod`) with a CIDR block sized for growth, e.g. `10.0.0.0/16`.
- Availability Zones: use 3 AZs when available to improve resilience.
- Subnet layout (per AZ):
  - Public Subnet (e.g. `10.0.X.0/24`): ALB, NAT Gateway elastic IPs (if required by architecture), bastion (optional). Only resources that must be internet-reachable.
  - Private Application Subnet (e.g. `10.0.(X+10).0/24`): EKS worker nodes / Fargate ENIs / ECS tasks.
  - Private Data Subnet (e.g. `10.0.(X+20).0/24`): RDS/Aurora instances and caches; tightly restricted inbound access.
  - Optional: Isolated Subnet for batch/backup jobs with no internet egress.
- Use subnet tags `kubernetes.io/cluster/<cluster-name>=shared` as required.

2) Routing & Gateways
- Internet Gateway (IGW) attached to VPC for public subnets.
- NAT Gateways in each AZ (or use AWS NAT Gateway High-Availability design) for private subnet outbound internet access. Prefer one NAT per AZ to avoid cross-AZ charges and reduce blast radius.
- Route Tables:
  - Public route table -> IGW
  - Private route table -> NAT Gateway (per AZ)
  - Data subnet route table -> no direct internet; restricted routes only
- Consider using AWS NAT Instances only when you need custom packet handling (rare).

3) VPC Endpoints & Private Connectivity
- Use interface VPC endpoints (AWS PrivateLink) for: `secretsmanager`, `ecr.api`, `ecr.dkr`, `sts`, `ssm`, `ssmmessages`, `ec2`, `s3-control` (if needed), `logs` (for CloudWatch Logs ingestion where supported).
- Use Gateway endpoints for: `s3`, `dynamodb` (reduces internet egress costs and improves security).
- Endpoint policies: apply least-privilege policies on endpoints where possible to restrict which principals or services can use them.
- Transit / inter-account networking:
  - Use AWS Transit Gateway for multi-account multi-VPC connectivity at scale.
  - For small setups, VPC Peering between environment VPCs is acceptable but becomes hard to manage at scale.
- Hybrid connectivity:
  - AWS Direct Connect or AWS VPN for private connectivity to on-prem. Use Direct Connect with Transit Gateway for predictable latency.

4) Security Groups (SG) — Principles
- Security Groups are stateful and are the primary in-VPC network access control.
- Keep SGs narrow and intent-based. Name pattern: `sg-<service>-<direction>-<env>`.
- Attach least-privilege rules using Service-to-Service access patterns.

Example SGs and rules
- `sg-alb-public` (ALB)
  - Inbound: `TCP 80/443` from 0.0.0.0/0 (or restrict to CloudFront / WAF IPs)
  - Outbound: `TCP 30000-32767` to `sg-eks-workers` (if using NodePort), `TCP 443` to egress endpoints
- `sg-eks-workers` (EKS worker nodes / pods via ENIs)
  - Inbound: `TCP 443` from `sg-alb-public` (target group), `TCP 10250` from `sg-controlplane` (if self-managed), `TCP` from other service SGs for specific APIs
  - Outbound: `TCP 5432` to `sg-rds`, `TCP 443` to S3 endpoints and `ECR` endpoints, `TCP` to `sg-sqs` if applicable
- `sg-rds` (database)
  - Inbound: `TCP 5432` from `sg-eks-workers` and `sg-ecs-workers` only
  - Outbound: minimal, typically to monitoring account endpoints
- `sg-redis` / `sg-elasticache`
  - Inbound: `TCP 6379` from application SGs only
- `sg-ci-cd` (CI runners)
  - Inbound: limited to management or ephemeral, outbound to ECR and AWS services

5) Network ACLs (NACLs)
- Use NACLs as additional coarse-grained protection at subnet level for defense-in-depth.
- Default allow/deny: prefer default `allow` NACLs for simplicity and rely on SGs. Use custom NACLs when you require subnet-level controls or to restrict known-bad source ranges.
- Keep NACL rules idempotent and documented.

6) EKS-specific networking
- Pod networking: use AWS VPC CNI (`amazon-vpc-cni-k8s`) for pod IPs in VPC (easier Security Group mapping via ENIs) or use Cilium for enhanced security features.
- Use IAM Roles for Service Accounts (IRSA) to grant pods AWS permissions with least privilege.
- Restrict `kubelet` and `kube-apiserver` access using security groups and endpoint private access where possible.
- Consider enabling `Private Cluster` mode (control plane not exposed publicly). Use AWS ALB Ingress Controller or AWS Load Balancer Controller with ALB in public subnets.

7) Secrets Management & Key Management
- Secrets: centralize in `AWS Secrets Manager` (recommended) or `SSM Parameter Store (SecureString)` for smaller teams.
- Encryption: Use `AWS KMS` CMKs per environment. Use separate KMS key for `prod` and `staging` to reduce blast radius.
- Secrets access: use IAM policies and IRSA to allow only the service principal/pod to access required secrets; do not use broad role access.
- Rotation: enable automatic rotation for DB credentials where supported; schedule rotations and ensure client support for reloading secrets.
- No secrets in container images, source code, or plaintext config files. Use environment injection at runtime (Kubernetes Secrets mounted via projected volumes with IRSA fetchers or use sidecar secret fetcher agents).

8) TLS / Certificate Management
- Use AWS Certificate Manager (ACM) for public certificates in `us-east-1` for CloudFront or regional ACM for ALB/API Gateway.
- Enforce TLS 1.2+ for all inbound and inter-service communication.
- Use mTLS inside the cluster for high security needs (service mesh: Istio/Linkerd/Consul) and sidecar TLS termination.

9) External Protection & Edge Controls
- Use `AWS WAF` in front of CloudFront/ALB for OWASP protections, IP reputation blocking, and rate limiting.
- Enable AWS Shield Advanced if DDoS risk is high for critical apps.
- Use CloudFront for caching, SSL termination, and geo-blocking rules.

10) IAM & Access Patterns
- Principle of least privilege for IAM users, roles and policies.
- Use `IAM Roles for Service Accounts (IRSA)` for pods to call AWS APIs.
- Avoid long-lived AWS access keys in CI; use OIDC for GitHub Actions to assume roles into AWS for ephemeral creds.
- Enforce MFA and SSO (AWS SSO / OIDC provider) for human user access.

11) Logging, Flow Logs & Monitoring
- Enable VPC Flow Logs (to CloudWatch or S3) for audit and network troubleshooting. Consider sampling if volume is high.
- Centralize logs into a logging account or centralized S3 bucket + CloudWatch logs cross-account subscription.
- Send CloudTrail logs to a centralized account and enable log file integrity validation.
- Enable GuardDuty, SecurityHub, and AWS Config for security posture and drift detection.

12) Observability & Alerting for Network/Security
- Monitor: NAT Gateway bandwidth, ELB 5xx rates, RDS connectivity errors, VPC Flow abnormal patterns.
- Set alerts for: public subnet resource creation, public S3 buckets, changes to Security Groups allowing 0.0.0.0/0 ports, KMS key policy changes.
- Integrate alerts into incident management (PagerDuty/Opsgenie) and include runbook links.

13) Operational Considerations
- Bastion access: prefer `AWS Systems Manager Session Manager` + SSM Agent instead of SSH bastion hosts. If SSH required, restrict to known IPs and ephemeral keys.
- Patch management: use AWS Systems Manager Patch Manager to keep AMIs/nodes updated.
- Rate-limits and quotas: track service quotas (ENI limits, NAT throughput) and request increases proactively before scale events.
- Cost: use VPC endpoints and NAT placement to reduce cross-AZ data costs; prefer S3 Gateway endpoints for heavy S3 traffic.

14) Compliance & Hardening
- CIS AWS Foundations benchmarks: implement account-level hardening and automated checks via Config rules.
- Disable or monitor unused open ports. Enforce encryption-at-rest and in-transit.

15) Example Terraform/networking checklist (high-level)
- `terraform apply` steps:
  - Create VPC, subnets, route tables, IGW, NATs
  - Create security groups and default deny rules
  - Create endpoint resources (`aws_vpc_endpoint` interface and gateway)
  - Provision GuardDuty, CloudTrail, Flow Logs
  - Set up Transit Gateway or peering as required
- Validate with automated tests: `terratest` or `kitchen-terraform` for smoke validation.

16) Quick Reference - Security Rules Example (Kubernetes app to DB)
- App SG -> DB SG: allow `TCP 5432` (Postgres) from `sg-app` only.
- App SG inbound: allow `TCP 443` from `sg-alb` only.
- DB SG inbound: block 0.0.0.0/0 and only allow known application SGs.

Summary
- Follow least-privilege network segmentation using security groups, VPC endpoints, and private subnets.
- Prefer SSM Session Manager for operator access; use IRSA for pod permissions; centralize logs and security tooling into a management account.
- Automate everything with IaC, build checks, and monitoring/alerts tied to runbooks.

Generated: 2025-12-05
