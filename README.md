# InfraOps — Scalable Web Service on AWS

**Live URL:** https://yulin-xia-bitgo-infra-web-services.com  
(`/health` · `/metrics`) · **Region:** `us-east-1` · **Cluster:** `bitgo-infraops-dev`

IaC (Terraform) and Kubernetes manifests are in this repo. Extended deploy notes, screenshots, and requirement traceability: [`docs/SUBMISSION_DETAILS.md`](docs/SUBMISSION_DETAILS.md).

---

## Architecture

![Architecture](docs/architecture.png)

**Flow:** Internet → Route53 → ALB (HTTPS, ACM) → Ingress → Service → Pods on EKS (private subnets, 2 AZs, NAT egress).

---

## What scales & what triggers it

| What scales | Trigger | Range |
|-------------|---------|--------|
| **Pod replicas** (HPA) | Average CPU **> 50%** of each pod’s `requests.cpu` (100m) | **2 → 10** |
| **EKS nodes** | Not auto-scaled in this submission (fixed node group 2–4) | — |

**Verified:** `hey -z 60s -c 30` on the live URL → 67,530× HTTP 200; HPA scaled **2 → 5** pods, then back to **2**. Evidence: [`docs/load-test-hey.png`](docs/load-test-hey.png), [`docs/hpa-scale-out.png`](docs/hpa-scale-out.png).

**HA:** ≥2 replicas across two AZs; ALB health check `/health` removes unhealthy targets (survives loss of one pod/node).

**Observability:** ALB CloudWatch dashboard `bitgo-infraops-dev-alb` + app `/metrics` (Prometheus).

**IAM:** IRSA for AWS Load Balancer Controller only; **no AWS keys** in the application Deployment.

---

## If I had another week

- **Cluster Autoscaler** when pods cannot schedule on existing nodes  
- **WAF** on the ALB  
- **S3 remote Terraform state** + DynamoDB locking  
- **Narrow IAM** policies instead of broad admin for Terraform  

---

## Cut for time (and why)

**Route53 + ACM: import into existing Terraform stacks** — Domain registration and ACM DNS validation were done in the console first so HTTPS was not blocked on certificate Pending; the resources are **managed in IaC** via `resources/route53` and `resources/acm` (imported into state, not “console-only”).

**Cluster Autoscaler (node autoscaling)** — Only **HPA** scales pods (2→10). Worker nodes stay on a **fixed** EKS node group (2–4 instances). Under a larger burst, new pods might not schedule if nodes run out of CPU/memory; I prioritized proving **pod-level** scale-out (PDF + `hey` test) within the time budget over adding node autoscaling and its IAM/scheduling wiring.

---

## Author

Yulin Xia — Fall 2026 BitGo InfraOps internship submission.
