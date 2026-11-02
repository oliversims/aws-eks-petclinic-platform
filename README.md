<h1 align="center">Petclinic Platform — AWS Infrastructure</h1>

<p align="center">
  <img src="docs/images/aws-main-logo-2025-orange-arrow-icon-1.png" alt="Amazon Web Services" width="140">
</p>

**DevOps Platform Portfolio** · **Oliver Sims**

[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC.svg)](https://www.terraform.io/)
[![AWS](https://img.shields.io/badge/Cloud-AWS%20us--east--1-232F3E.svg)](https://aws.amazon.com/)
[![EKS](https://img.shields.io/badge/Orchestration-Amazon%20EKS-FF9900.svg)](https://aws.amazon.com/eks/)
[![Helm](https://img.shields.io/badge/Packaging-Helm-0F1689.svg)](https://helm.sh/)
[![Argo CD](https://img.shields.io/badge/CD-Argo%20CD-EF7B4D.svg)](https://argo-cd.readthedocs.io/)
[![Karpenter](https://img.shields.io/badge/Autoscaling-Karpenter-FFA500.svg)](https://karpenter.sh/)

| **Region** | **DNS zone** | **Environments** |
|------------|--------------|------------------|
| `us-east-1` | `simsoliver.com` | `dev` · `prod` |

> **Note:** This repository is the public portfolio write-up. Application source, Terraform, Helm, Kubernetes manifests, and operator scripts remain in **private** repositories and are available on request. After end-to-end validation, **all AWS infrastructure was destroyed** to avoid ongoing EKS, NAT, and ALB cost. Screenshots below are from that live run.

---

## Contents

1. [Overview](#1-overview)
2. [Two-repository model](#2-two-repository-model)
3. [AI-assisted delivery](#3-ai-assisted-delivery-claude-code)
4. [Platform architecture](#4-platform-architecture)
5. [Application service topology](#5-application-service-topology)
6. [Infrastructure as code](#6-infrastructure-as-code)
7. [Continuous integration](#7-continuous-integration)
8. [Continuous delivery (GitOps)](#8-continuous-delivery-gitops)
9. [Production resilience](#9-production-resilience)
10. [Observability](#10-observability)
11. [Application UI](#11-application-ui-captured-while-running)
12. [Day-2 operations](#12-day-2-operations)
13. [Security practices](#13-security-practices)
14. [Environment comparison](#14-environment-comparison)
15. [Outcomes](#15-outcomes)

---

## 1. Overview

This portfolio is a production-shaped AWS platform for the Spring Petclinic microservices suite on Amazon EKS. It covers the work a DevOps / platform engineer owns day to day: infrastructure as code, secure networking, container delivery, GitOps continuous delivery, observability, and operational tooling.

A deliberate part of the build was **AI-assisted platform engineering** with Claude Code — guided by a Jira-style backlog and technical specification, with project rules and safety hooks under `.claude/` (see [§3](#3-ai-assisted-delivery-claude-code)).

The application is eight Spring Boot services (Spring Cloud Config, Eureka, API gateway, domain services, GenAI, and Spring Boot Admin). Delivery uses private ECR images, Helm, and Argo CD. CI never runs `kubectl apply` or `helm upgrade` against the cluster.

| Layer | Choice |
|-------|--------|
| Cloud | AWS (`us-east-1`) |
| IaC | Terraform (S3 + DynamoDB remote state) |
| Compute | Amazon EKS 1.34 · ARM (`t4g.medium`) · Karpenter |
| Packaging | Helm (one shared chart, per-service + per-env values) |
| Registry | Amazon ECR (`petclinic-dev/*` and `petclinic-prod/*`) |
| Data | Amazon RDS MySQL 8 (shared `petclinic` database) |
| Secrets | AWS Secrets Manager + External Secrets Operator |
| Edge | ALB · ACM · Route 53 · ExternalDNS · prod WAF |
| CI | GitHub Actions (OIDC to AWS) |
| CD | Argo CD ApplicationSets (auto-sync, prune, selfHeal) |
| Observability | Prometheus · Grafana · Alertmanager · Loki · Fluent Bit · Zipkin · Slack |

---

## 2. Two-repository model

Application and platform change at different rates, so delivery is split across two repositories.

| Repository | Responsibility |
|------------|----------------|
| `spring-petclinic-microservices` (private) | Java services, Docker images, `build-push.yml` (CI only) |
| `petclinic-platform` (private) | Terraform, Helm values, Argo CD, security policies, scripts, documentation |

**Delivery flow:** push to the app repo → build, scan, and push images to ECR → `repository_dispatch` to the platform repo → commit updated `image.tag` values → Argo CD reconciles the cluster from Git.

Folder layout of the private repositories (directories only):

```
petclinic-platform/
├── terraform/                    # Infrastructure as Code
│   ├── environments/
│   │   ├── dev/                  # Dev environment root module
│   │   └── prod/                 # Prod environment root module
│   └── modules/                  # Reusable modules
│       ├── vpc/                  # VPC, public+private subnets, IGW, NAT (1 dev / 2 prod), SGs
│       ├── eks/                  # EKS cluster, node groups, OIDC, IAM
│       ├── ecr/                  # ECR repos (per service per env), lifecycle policies
│       ├── rds/                  # RDS MySQL, subnet group, parameter group
│       ├── dns/                  # Route 53, ACM certificates
│       ├── secrets/              # Secrets Manager resources
│       ├── observability/        # Prometheus, Grafana, CloudWatch, FluentBit
│       ├── karpenter/            # NodePools, EC2NodeClass, on-demand ARM
│       └── github-oidc/          # GitHub Actions OIDC role (dev only, account-scoped)
├── k8s/
│   ├── argocd/                   # ApplicationSets, projects, install
│   ├── security/                 # NetworkPolicy, ResourceQuota (dev / prod)
│   ├── karpenter/                # NodePool and EC2NodeClass manifests
│   ├── k8sgpt/                   # Cluster insights (day-2)
│   ├── base/                     # Namespaces, External Secrets
│   └── observability/            # Prometheus rules, Grafana dashboards
├── helm/
│   └── petclinic-service/        # Generic chart (shared by all 8 services)
├── helm-values/                  # Per-service YAML + per-env (dev.yaml, prod.yaml)
├── .github/workflows/            # Platform CI (Argo CD handles CD)
├── .claude/                      # Rules, hooks, agents, skills
├── CLAUDE.md                     # Session brief
├── scripts/
│   ├── bootstrap/                # One-time TF state backend (S3 + DynamoDB)
│   ├── helm/                     # Helm lint / template / dry-run
│   ├── k8s/                      # apply / delete / clean leftovers
│   ├── env/                      # status / stop / start
│   ├── checks/                   # AZ spread checks
│   └── terraform/                # Terraform output / value checks
└── docs/                         # Architecture + ADRs

spring-petclinic-microservices/
├── spring-petclinic-*/           # Eight Spring Boot services
└── .github/workflows/            # build-push.yml (CI only)
```

---

## 3. AI-assisted delivery (Claude Code)

This portfolio also shows a disciplined way to use AI on infrastructure work: a backlog and technical specification as the contract, Claude Code as the implementer, and human review on every plan and apply.

| ![Claude](docs/images/claude.png) |
|:---:|
| Claude Code — used to implement platform stories under a fixed backlog and guardrails |

### Spec-driven workflow

Two documents steer almost every change:

| Document | Role |
|----------|------|
| `docs/jira-backlog.md` (private platform repo) | Delivery backlog: epics E-0…E-18, story IDs, acceptance criteria, and dependencies (~110 stories). Implementation proceeds **one story at a time**. |
| `docs/technical-spec.md` (private platform repo) | Source of truth for CIDRs, instance sizes, naming, IRSA, Helm/Argo layout, and security controls. Stories link to spec sections so values are not invented mid-session. |

**Working model**

1. Select the next `PETPLAT-xxx` story from the backlog  
2. Read the linked section of `technical-spec.md` before editing Terraform, Helm, or Kubernetes manifests  
3. Implement only what the acceptance criteria require  
4. Review the diff, run plan/validate scripts, then apply and verify in AWS  

Epic **E-0 (Claude Code Setup)** was completed first so the AI environment — rules, hooks, and agents — was productized before VPC and EKS work began.

### Session brief: `CLAUDE.md`

Root `CLAUDE.md` (private platform repo) is the standing brief for every session: repository layout, Terraform/Helm/Argo conventions, non-negotiable security rules, environment differences, CI/CD boundaries, and available MCP tools. That keeps implementation consistent across sessions.

### `.claude/` configuration

| Piece | Purpose |
|-------|---------|
| **Rules** (`.claude/rules/`) | Path-scoped standards for Terraform, Helm, Argo CD, DNS, observability, Karpenter, security, pipelines, and docs |
| **Hooks** (`.claude/hooks/` + `settings.json`) | Safety controls: block reckless destroy/`rm -rf` on infra paths, warn on apply without `plan.out`, block committing secrets |
| **Agents** (`.claude/agents/`) | Read-only reviewers: `terraform-reviewer`, `security-auditor`, `pipeline-reviewer`, `k8s-validator` |
| **Skills** (`.claude/skills/`) | Repeatable ops playbooks: `/terraform-plan`, `/terraform-apply`, `/security-scan` |
| **MCP** | Terraform Registry docs, AWS knowledge, pricing estimates, library docs, and ticket-driven flow |

The skill demonstrated here is **designing context, constraints, and review loops** so an assistant produces production-shaped infrastructure instead of one-off scripts.

---

## 4. Platform architecture

End-to-end view of CI/CD, the AWS account boundary, VPC layout, EKS namespaces, data stores, and observability.

![Platform architecture overview](docs/images/arch-overview-step42-hires.png)

**Highlights**

- Separate CI (app repo) and CD (platform repo + Argo CD)
- Public subnets for IGW, NAT, and ALBs; private subnets for EKS and RDS
- Managed node group plus Karpenter for additional capacity
- Dedicated namespaces for apps, Argo CD, monitoring, tracing, External Secrets, and Karpenter
- Production edge: WAF on the application ALB and a dedicated ops ALB

---

## 5. Application service topology

Inbound traffic terminates TLS on the ALB and reaches `api-gateway`. Domain services register with Eureka. Only customers, visits, and vets talk to MySQL. GenAI calls OpenAI over NAT. Selected services export traces to Zipkin.

![Service topology](docs/images/service-diagram.png)

| Service | Port | Role |
|---------|------|------|
| config-server | 8888 | Central configuration (starts first) |
| discovery-server | 8761 | Eureka registry |
| api-gateway | 8080 | Public entry and routing |
| customers-service | 8081 | Owners and pets (MySQL) |
| visits-service | 8082 | Visit records (MySQL) |
| vets-service | 8083 | Veterinarians (MySQL) |
| genai-service | 8084 | Chat / AI (OpenAI) |
| admin-server | 9090 | Spring Boot Admin |

Startup order is enforced with init containers: config → discovery → remaining services.

---

## 6. Infrastructure as code

Terraform roots under `terraform/environments/{dev,prod}` compose reusable modules:

`vpc` · `eks` · `ecr` · `rds` · `secrets` · `dns` · `observability` · `karpenter` · `github-oidc`

Each environment has its own state key (`petclinic/{env}/terraform.tfstate`). Sensitive inputs (for example OpenAI and Slack webhook URLs) are supplied at plan/apply time via `TF_VAR_*` and are never committed.

### Network

- Workloads in private subnets have no public IPs
- Dev: one NAT gateway · Prod: NAT gateway per AZ
- Security groups allow RDS `3306` from the node security group only
- ALB is the only intended public ingress (`0.0.0.0/0` on 80/443)

### Compute and data

- EKS **1.34** with IRSA for controllers
- Managed nodes: **3 × t4g.medium** (ARM64)
- Karpenter NodePool for on-demand `t4g.medium` / `t4g.large`
- RDS MySQL 8: `db.t4g.micro` (dev) · `db.t4g.small` (prod), single-AZ, encrypted

### Proven in AWS

| ![AWS IAM console sign-in](docs/images/aws-login.png) |
|:---:|
| AWS console sign-in — operator access; CI authenticates with GitHub OIDC |

| ![Managed node group plus Karpenter nodes in EC2](docs/images/aws-ec2-eks-nodes-managed-and-karpenter.png) | ![EKS Compute tab — all nodes Ready](docs/images/eks-compute-nodes-ready.png) |
|:---:|:---:|
| Six `t4g.medium` instances across `us-east-1a/b/c`: managed EKS + Karpenter | EKS Compute: managed node group and Karpenter Ready |

### Container registry (ECR)

![ECR repositories for petclinic-prod](docs/images/aws-ecr-petclinic-prod.png)

*Private ECR repositories under the `petclinic-prod` prefix.*

### Live cluster workloads (EKS console)

Platform add-ons, application pods, and K8sGPT running on the validated production cluster:

| ![Platform add-on pods (Argo CD, Karpenter, External Secrets)](docs/images/eks-pods-platform-addons-a.png) | ![Platform add-on pods (Argo CD, AWS Load Balancer Controller, EBS CSI)](docs/images/eks-pods-platform-addons-b.png) |
|:---:|:---:|
| Argo CD · Karpenter · External Secrets | Argo CD · AWS Load Balancer Controller · EBS CSI |
| ![Platform add-on pods (ApplicationSet, ExternalDNS, Fluent Bit)](docs/images/eks-pods-platform-addons-c.png) | ![Application pods with node agents](docs/images/eks-pods-apps-and-daemonsets.png) |
| ApplicationSet · ExternalDNS · Fluent Bit | App pods with node agents |
| ![K8sGPT operator with application pods](docs/images/eks-pods-k8sgpt-and-apps-a.png) | ![K8sGPT analyzer and application pods across another AZ](docs/images/eks-pods-k8sgpt-and-apps-b.png) |
| K8sGPT operator with application pods | K8sGPT analyzer across another AZ |

---

## 7. Continuous integration

In the application repository, `build-push.yml` is run from **Actions → Run workflow** (`workflow_dispatch`). That path builds all eight services. Path filters are in the workflow for when push-to-main is enabled.

1. Detect services (all eight on a manual run)  
2. Matrix-build ARM64 images  
3. Scan with Trivy (CRITICAL/HIGH uploaded as artifacts; the scan does not fail the job)  
4. Push to `petclinic-prod/*` using the short commit SHA  
5. Dispatch `app-image-built` to the platform repository  

AWS authentication uses GitHub OIDC — no long-lived access keys in the workflow.

| ![GitHub login](docs/images/github-login.png) |
|:---:|
| GitHub — CI identity provider (OIDC; no long-lived AWS keys) |

| ![GitHub Actions matrix build in progress](docs/images/github-actions-build-push-running.png) | ![GitHub Actions successful build and platform dispatch](docs/images/github-actions-build-push-success.png) |
|:---:|:---:|
| Matrix ARM64 build in progress | Successful build and `app-image-built` dispatch |

The platform workflow `update-image-tags.yml` receives the dispatch, updates only the listed services’ Helm values with `yq`, and commits so Argo CD can reconcile.

---

## 8. Continuous delivery (GitOps)

Argo CD is installed per cluster. One ApplicationSet per environment renders eight Applications from Helm (`helm/petclinic-service` + `helm-values/{service}.yaml` + `helm-values/{env}.yaml`).

Both environments use **automated sync** with **prune** and **selfHeal**. CI never runs `kubectl` or `helm upgrade`.

| ![Argo CD login — argocd.simsoliver.com](docs/images/argocd-login.png) | ![Argo CD Applications — petclinic-prod](docs/images/argocd-petclinic-prod-apps.png) |
|:---:|:---:|
| Argo CD login (`argocd.simsoliver.com`) | All eight production applications Healthy and Synced |

### Per-service Application trees (production)

| ![admin-server-prod](docs/images/argocd-admin-server-prod.png) | ![api-gateway-prod](docs/images/argocd-api-gateway-prod.png) |
|:---:|:---:|
| admin-server | api-gateway |
| ![config-server-prod](docs/images/argocd-config-server-prod.png) | ![customers-service-prod](docs/images/argocd-customers-service-prod.png) |
| config-server | customers-service |
| ![discovery-server-prod](docs/images/argocd-discovery-server-prod.png) | ![genai-service-prod](docs/images/argocd-genai-service-prod.png) |
| discovery-server | genai-service |
| ![vets-service-prod](docs/images/argocd-vets-service-prod.png) | ![visits-service-prod](docs/images/argocd-visits-service-prod.png) |
| vets-service | visits-service |

Production defaults to three replicas for traffic-facing services, with topology spread (`maxSkew: 1`, `DoNotSchedule`) so pods land across availability zones. Discovery and admin remain at one replica by design.

---

## 9. Production resilience

Multi-AZ scheduling is verified with an operational script rather than assumed from Helm values alone.

| ![AZ spread check output](docs/images/kubectl-az-spread-check.png) | ![AZ spread result OK](docs/images/kubectl-az-spread-ok.png) |
|:---:|:---:|
| Zone placement check | Result: OK |

**Additional production controls**

- Horizontal Pod Autoscaler on request-path services
- PodDisruptionBudgets on critical workloads
- NetworkPolicies (default deny in the application namespace)
- ResourceQuota and LimitRange
- AWS WAF on the production application ALB
- Separate ops ALB for Grafana, Zipkin, and Argo CD in production

---

## 10. Observability

The observability stack is installed with Terraform `helm_release` into `monitoring` and `tracing` (not managed by Argo CD ApplicationSets).

| Signal | Tooling |
|--------|---------|
| Metrics | kube-prometheus-stack → Grafana |
| Logs | Fluent Bit → Loki → Grafana Explore |
| Traces | Zipkin (gateway, customers, visits, vets, genai) |
| Alerts | Alertmanager → Slack |
| Cluster insights | K8sGPT → Slack (dedicated channel) |

### Metrics and Kubernetes views

| ![Grafana login](docs/images/grafana-login.png) |
|:---:|
| Grafana — metrics and Kubernetes dashboards |

| ![JVM metrics dashboard](docs/images/grafana-jvm-metrics.png) | ![Namespace compute resources](docs/images/grafana-k8s-compute-namespace.png) |
|:---:|:---:|
| JVM metrics | Namespace compute |
| ![Namespace compute and network table](docs/images/grafana-k8s-compute-network.png) | ![Networking overview](docs/images/grafana-k8s-networking-overview.png) |
| Compute and network table | Networking overview |

![Networking time series](docs/images/grafana-k8s-networking-graphs.png)

*Networking time series.*

### Logs (Loki)

| ![Loki log volume](docs/images/grafana-loki-explore-volume.png) | ![config-server logs in Loki](docs/images/grafana-loki-config-server-logs.png) | ![Pod log explore](docs/images/grafana-loki-pod-logs.png) |
|:---:|:---:|:---:|
| Log volume | config-server logs | Pod explore |

### Traces (Zipkin)

![Zipkin trace list](docs/images/zipkin-traces-list.png)

*Zipkin traces from gateway, customers, visits, vets, and genai.*

### Alerting

| ![Slack sign-in](docs/images/slack-signin.png) |
|:---:|
| Slack — destination for Alertmanager and K8sGPT notifications |

| ![Alertmanager Slack alerts](docs/images/slack-alertmanager-grafana.png) | ![K8sGPT Slack insights](docs/images/slack-petclinic-alerts-k8sgpt.png) |
|:---:|:---:|
| Alertmanager → Slack | K8sGPT → Slack |

### K8sGPT

K8sGPT is the AI diagnosis path for the **application namespace**, separate from Prometheus and Alertmanager. Alertmanager fires on metrics and posts to one Slack channel. K8sGPT watches Kubernetes objects, asks OpenAI to explain issues, and posts plain-language findings to a **second** Slack channel.

It is optional day-2 tooling: `scripts/k8s/apply-k8s.sh` installs it after the cluster and applications are healthy (STEP-13 through STEP-15). Manifests live under `k8s/k8sgpt/`; the operator is not an Argo CD Application.

**How it is wired**

- Helm installs the operator into `k8sgpt-operator-system`
- External Secrets Operator copies the existing GenAI OpenAI key (`petclinic/prod/openai-api-key`) into Secret `k8sgpt-openai` — no second API key
- Custom Resource `petclinic` scopes analysis to namespace `petclinic-prod`, every **5 minutes**, with OpenAI **gpt-4o-mini**
- Analyzer whitelist: Pod, Deployment, ReplicaSet, PersistentVolumeClaim, Service, Ingress, HorizontalPodAutoscaler (ops namespaces are excluded)
- Additional RBAC (`rbac-ingressclass.yaml`) allows accurate IngressClass checks
- Findings are stored as Result custom resources and sent to Slack via `spec.sink` (dedicated webhook, not Alertmanager)

**Full flow**

```
AWS Secrets Manager
petclinic/prod/openai-api-key
        │
        │  External Secrets Operator
        ▼
Secret k8sgpt-openai
(ns: k8sgpt-operator-system)
        │
        ▼
K8sGPT operator (Helm)
watches CR "petclinic" every 5m
        │
        ├─ scan ──────►  namespace petclinic-prod
        │                Pod · Deployment · ReplicaSet
        │                PVC · Service · Ingress · HPA
        │
        ├─ explain ───►  OpenAI (gpt-4o-mini)
        │
        ├─ store ─────►  Result CRs (operator namespace)
        │
        └─ notify ────►  Slack (k8sgpt channel)
                         (not the Alertmanager channel)


Install order (apply-k8s.sh)

  STEP-13  install-operator.sh
  STEP-14  openai-secret.yaml
  STEP-15  k8sgpt.yaml  +  rbac-ingressclass.yaml
              │
              ▼
         operator loop:  scan  →  AI explain  →  Result + Slack
```

---

## 11. Application UI (captured while running)

While the clusters were up, users reached the UI over HTTPS on `simsoliver.com`, with GenAI chat in the same experience. **Those endpoints are not live now** — the environment was torn down afterward for cost control. The screenshots below document the working application from that period.

| ![Petclinic home with GenAI chat](docs/images/petclinic-ui-home-chat.png) |
|:---:|
| Home + GenAI chat |

| ![Owners list](docs/images/petclinic-ui-owners-list.png) | ![Owner details with chat](docs/images/petclinic-ui-owner-details-chat.png) |
|:---:|:---:|
| Owners list | Owner details + chat |

**Hostnames used during the validated run** (not currently served)

- App: `petclinic-dev.simsoliver.com` · `petclinic.simsoliver.com`
- Ops: Grafana, Zipkin, and Argo CD under the same zone (shared ALB in dev; dedicated ops ALB in prod)

---

## 12. Day-2 operations

Scripts under `scripts/` support lifecycle work without reinventing Terraform:

| Script | Purpose |
|--------|---------|
| `bootstrap/bootstrap-state.sh` | One-time S3 + DynamoDB state backend |
| `k8s/apply-k8s.sh` / `delete-k8s.sh` / `clean-leftovers.sh` | Cluster YAML apply, teardown, leftover ALB/EBS cleanup |
| `k8s/verify-k8s.sh` | Production post-apply verification |
| `helm/validate-helm.sh` | Lint and template all service × env combinations |
| `env/env-status.sh` / `start-env.sh` / `stop-env.sh` | Pause and resume cost-sensitive capacity |
| `checks/check-az-spread.sh` | Verify production replica zone spread |
| `terraform/verify-tf.sh` | Terraform output / value checks |

Operator runbooks (`dev-deploy.txt`, `prod-deploy.txt`, `destroy-env.txt`) document the apply and teardown order.

---

## 13. Security practices

- Secrets in AWS Secrets Manager; synced into the cluster with External Secrets Operator
- GitHub Actions authenticates with OIDC (no long-lived AWS access keys)
- Private subnets for nodes and RDS; encryption at rest for RDS and ECR
- Image scanning in CI (Trivy CRITICAL/HIGH reports as workflow artifacts)
- Least-privilege IAM for IRSA-backed controllers
- NetworkPolicy default deny in application namespaces
- Production WAF on the public application load balancer
- Sensitive Terraform inputs via `TF_VAR_*`; secret-bearing variable files are gitignored

---

## 14. Environment comparison

| Concern | Dev | Prod |
|---------|-----|------|
| VPC CIDR | `10.0.0.0/16` | `10.1.0.0/16` |
| Availability zones | 2 | 3 |
| NAT | 1 | 1 per AZ |
| Node desired | 3 × `t4g.medium` | 3 × `t4g.medium` |
| App replicas | 1 | 3 (discovery & admin = 1) |
| Topology spread | Preferred (soft) | `DoNotSchedule`, maxSkew 1 |
| ALB | Shared (app + ops) | App ALB + ops ALB |
| WAF | — | On app ALB |
| RDS | `db.t4g.micro` | `db.t4g.small` |
| ECR tags | Mutable | Immutable |
| Argo CD sync | Auto prune + selfHeal | Auto prune + selfHeal |

---

## 15. Outcomes

What this portfolio demonstrates:

1. **Commit-to-cluster delivery** across two repositories with OIDC and GitOps  
2. **Production-shaped EKS** on Graviton with Karpenter, multi-AZ pod spread, HPA, and PDB  
3. **In-cluster observability** — metrics, logs, traces, and Slack alerting  
4. **Operational maturity** — validation scripts, AZ checks, env pause/resume, and clean teardown helpers  
5. **Security defaults** — private networking, Secrets Manager, image scanning, NetworkPolicy, and WAF  
6. **AI-assisted platform engineering** — backlog and technical spec as the contract, Claude Code rules/hooks/agents as the execution environment, human review on the critical path  

---

*Oliver Sims · Petclinic AWS EKS platform portfolio*
