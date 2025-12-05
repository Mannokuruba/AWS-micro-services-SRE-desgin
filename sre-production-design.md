**SRE Production Design — AWS Microservices Architecture**

Purpose
- **Goal:** Provide an SRE-oriented production deployment plan for the AWS microservices architecture: reliable, observable, secure, and automatable.
- **Scope:** Production deployment & operations patterns: CI/CD, releases, SLOs/SLIs, observability, security, DR, runbooks, and scaling.

**Architecture Summary:**
- **Ingress:** Route53 -> CloudFront -> WAF -> ALB / API Gateway (regional). Public subnets host agility components (ALB, NAT). Private subnets host EKS (microservices), ECS/Fargate (workers), RDS/Aurora (primary + read replicas), DynamoDB, S3, SQS/SNS/EventBridge, Lambda for event handlers. ECR for images, Secrets Manager + KMS for secrets.

**Production Environment Strategy**
- **Environments:** `dev` (short-lived), `staging` (production-like, test deployment), `prod` (single authoritative). Mirror prod VPC/subnet topology in staging with lower capacity.
- **Account strategy:** Use AWS Organizations — separate accounts for `management`, `networking` (shared VPC), `platform` (CI/CD, registry), `prod` workloads, and `sandbox/dev`. Centralize logging, security tooling in management account.
- **Regions & Availability Zones:** Multi-AZ mandatory. Consider active-active multi-region for critical services; otherwise active-passive for DR.

**Infrastructure as Code & Change Management**
- **IaC:** Terraform for infra (modules per service concern), Helm charts for Kubernetes deployments. Keep module versions pinned and tested. Store state in remote backend (`s3` + DynamoDB lock) with strict RBAC.
- **Git workflow:** GitHub/GitLab mainline with feature branches. Protected branches: `main`, `release/*`. PRs required with code owners. All changes to infra and manifests via GitOps or pipeline.
- **Change windows & approvals:** Emergency fixes have defined escalation. Non-emergency infra changes require peer review + automated plan/apply preview.

**CI/CD & Release Management**
- **Pipeline model:** Build -> Test -> Push image to `ECR` -> Publish artifact -> Deploy.
  - Build run in ephemeral runners. Images scanned with Clair/Trivy. Signed SBOM attached to image.
  - Unit -> integration -> contract tests run in pipeline; integration tests run in isolated ephemeral infra.
- **Deployment orchestration:** Use GitOps (ArgoCD) to reconcile manifests in `staging` and `prod`. Pipelines push to a `manifests` repo or tag to trigger ArgoCD sync.
- **Release strategies:** Prefer automated canary deployments with metric-based promotion. Fallback to blue/green for large schema/data changes. Enable quick rollback via GitOps or image tag revert.
- **Feature flags:** Use feature flag system (LaunchDarkly, OpenFeature + in-house) for progressive enablement.

**Deployment Safety & DB Migrations**
- **Backward-compatible migrations:** Always implement expands-before-contracts. Use double-write / read-from-old-and-new pattern for risky migrations.
- **Migration windows:** Coordinate schema changes with ops; run non-blocking migrations during low traffic when possible.
- **Schema migration tooling:** Use versioned migrations (Flyway/Liquibase/gh-ost) and run them in a controlled job with a dry-run mode.

**Autoscaling & Capacity**
- **Kubernetes autoscaling:** Horizontal Pod Autoscaler (HPA) driven by CPU/memory/custom metrics (queue depth, reqs/s, latency). Use Cluster Autoscaler for node scaling; prefer managed node groups or Fargate to reduce ops.
- **Worker scaling:** SQS-based workers scale on queue length with a safety cap and rate limits.
- **Capacity planning:** Monthly review using traffic growth forecasts and load test data. Keep buffer capacity (headroom) in node pools for spikes.

**Reliability SLOs / SLIs / Error Budgets**
- **Example SLOs:**
  - **API success SLO:** 99.95% availability (4.38 min downtime/month) for core public API (P99 latency < 500ms for 95% of requests).
  - **Background job throughput SLO:** 99.9% of jobs processed within SLA (e.g., 1 hour).
  - **Data durability SLO (RDS):** Backups success 100% daily; restore tested quarterly.
- **SLI sources:** CloudWatch metrics, custom metrics (Prometheus/OpenTelemetry), request tracing (X-Ray/OpenTelemetry), logs.
- **Error budget policy:** If error budget depleted (> threshold), freeze non-critical changes and run remediation until budget restored.

**Observability & Monitoring**
- **Metrics:** Prometheus/OpenTelemetry -> CloudWatch Container Insights / Managed Prometheus. Collect app-level (requests, errors, latency), infra-level (CPU, mem, disk), business metrics.
- **Logging:** Structured JSON logs, centralized shipping to CloudWatch Logs + long-term S3 bucket with lifecycle rules. Use query indexers (CloudWatch Logs Insights) and retain recent logs for fast lookup.
- **Tracing:** Integrate OpenTelemetry with traces sent to AWS X-Ray or Jaeger (managed). Trace cold paths and tail-sampling for high-volume services.
- **Dashboards & Alerting:** Grafana + CloudWatch dashboards with runbook links in alerts.
- **Alerting policy:** Alert on symptoms, not root cause. Severity: P1 (SEV), P2, P3. Use alert grouping and suppression windows to avoid alert storms.

**Incident Response & Runbooks**
- **On-call rotations:** Small rotation teams with SRE and service owners. Use escalation policies (PagerDuty/Opsgenie).
- **Runbooks:** For each critical path create a runbook with: symptoms, probable causes, quick mitigation steps, deeper remediation, rollback steps, postmortem template.
- **Post-incident:** Blameless postmortem within 72 hours, assign action items, track to completion.
- **Playbook example (deployment failure):**
  - Detect via high error % or latency.
  - Auto-triage: Run smoke tests; if failing, pause/stop rollout (ArgoCD pause or cancel) and rollback to previous image.
  - Notify channel + paging if P1.
  - If DB related, stop write traffic with feature flags, failover to read replica if needed.

**Security & Compliance**
- **IAM:** Least privilege roles using IAM roles for service accounts (IRSA) for EKS, fine-grained roles per service.
- **Network:** Private subnets for compute, use VPC endpoints for S3/DynamoDB/ECR/Secrets Manager. WAF rules tuned for OWASP top-10 protections.
- **Secrets:** Store secrets in AWS Secrets Manager / Parameter Store encrypted with KMS. Rotate credentials periodically. No secrets in code or images.
- **Image security:** Scanning on build, immutable tags, image signing, vulnerability triage process.
- **Pen tests & audits:** Quarterly vulnerability scans and annual penetration testing.

**Backup & Disaster Recovery**
- **Backups:** Automated RDS snapshots (automated + manual before risky changes). S3 lifecycle + cross-region replication for critical buckets.
- **Recovery objectives:** Define RTO/RPO per workload. For user-critical services target RTO < 15 minutes; for low-priority services RTO < 4 hours.
- **DR exercises:** Run tabletop and live failover drills yearly (and after major infra changes).
- **Multi-region considerations:** Active-active for read-heavy services (DynamoDB global tables), or active-passive with RDS cross region read replicas + promoted failover.

**Testing & Validation**
- **Test pyramid:** Unit tests, integration tests, contract tests, E2E, load/perf testing.
- **Pre-deploy checks:** Automated smoke tests run in `staging`. Canary health checks (latency, error rate, business metrics) govern promotion.
- **Chaos engineering:** Periodic chaos experiments (kill pods, simulate AZ failure) in staging, limited experiments in prod during maintenance windows.

**Cost Management**
- **Tagging policy:** Enforce tags (owner, team, cost-center, environment) on all resources, validated by pipeline.
- **Budgets & alerts:** AWS Budgets with Slack alerts. Rightsize instances based on utilization data, use Savings Plans for steady workloads.

**Operational Automation & Tooling**
- **Self-service platform:** Expose standard templates (Terraform modules, Helm charts) so dev teams can request infra safely.
- **On-call tooling:** Integrate runbooks in alert notifications, add quick actions (panic switch rollback) in runbook.
- **Security automation:** Auto-remediation playbooks for common issues (unencrypted S3, public AMIs).

**Example Runbook — Rolling Back a Bad Release**
1. **Symptom:** Errors > threshold or failed smoke tests in canary.
2. **Immediate actions:** Pause rollout; mark release as failed in CI/CD; alert on-call.
3. **Mitigation:** Revert Kubernetes Deployment to `image: previous-tag` via GitOps (create PR to `manifests` repo with previous image tag) or use `kubectl rollout undo` if manual remediation required.
4. **Verification:** Run smoke tests against health endpoints, verify metrics returned to baseline.
5. **Post-action:** Create incident ticket, capture timeline, and run a postmortem.

**Sample CI/CD flow (high-level)**
- Commit -> CI builds image -> Run unit + static analysis -> Push to ECR with digest and tag -> Run integration tests in ephemeral cluster -> Push manifest change PR to `manifests` repo (image digest) -> ArgoCD auto-applies to staging -> Canary deploy to prod via staged promotion -> Monitor canary metrics -> Full promotion.

**Appendix — Examples & Templates**
- Use Terraform modules: `vpc/`, `eks/`, `rds/`, `ecr/`, `iam/` with well-defined inputs and outputs.
- `sre-production-design.md` stored in repo root for visibility.

**Next steps & recommendations**
- Implement minimal IaC scaffold (VPC + EKS + ALB) in Terraform and test end-to-end in `staging` account.
- Create a minimal `post-deploy` smoke test suite and hook it to the pipeline to gate production promotions.
- Draft runbooks for top-5 most likely incidents (API outage, DB outage, high latency, queue backlog, deployment failure).


---

Generated: 2025-12-05
