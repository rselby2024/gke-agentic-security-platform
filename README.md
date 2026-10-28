# Governed Multi-Agent Security Platform on GKE

Public portfolio repository: https://github.com/rselby2024/gke-agentic-security-platform

A cloud-native, zero-trust security automation platform on Google Kubernetes Engine (GKE) Autopilot. It detects, investigates, governs, and remediates threats across Kubernetes workloads, with a human approval gate before any controlled action.

This repository is being built in small pieces. The directory layout is in place. Implementation lands only as each piece is approved, and this README is updated to match.

## Anchor mental model

```text
OS eBPF -> Node COS -> Pod IP -> NetworkPolicy Deny -> VPC Private -> Managed Service -> Back
```

Traffic and trust stay inside that path: kernel telemetry on Container-Optimized OS, pod identity, default-deny network policy, a private VPC, and managed Google APIs reached through Private Google Access.

## Architecture

Event-driven agents share one path from signal to evidence:

1. **Runtime and scanning telemetry.** Falco (eBPF kernel syscalls), Trivy (vulnerability scanning), and Prowler (CSPM) emit findings.
2. **Event bus.** Pub/Sub carries events on 8 isolated topics, each with its own dead-letter queue. Publishers and subscribers reach Pub/Sub through Private Google Access (`199.36.153.8/30:443`).
3. **AI investigation.** The IR Analyst agent calls Vertex AI (Gemini 2.5) and must return JSON that matches a strict response schema.
4. **Governance.** An Open Policy Agent (OPA) Rego sidecar evaluates each decision as `DENY`, `PERMIT`, or `REQUIRE_APPROVAL`.
5. **Human approval.** Approvals are signed with Cloud KMS asymmetric keys (`EC_SIGN_P256_SHA256`). Nonce and expiration checks reject replayed approvals.
6. **Controlled execution.** An MCP Gateway (mTLS on port 8443) fronts an MCP Server (FastMCP). Kubernetes RBAC for remediation is limited to the `app01` namespace.
7. **Audit and evidence.** Firestore holds workflow state. BigQuery dataset `agentic_governance_evidence` stores the immutable audit trail.

```text
Falco / Trivy / Prowler
        |
     Pub/Sub
        |
  IR Analyst (Vertex AI, schema-bound JSON)
        |
  OPA  -- DENY | PERMIT | REQUIRE_APPROVAL
        |
  Approval (Cloud KMS signature, nonce, expiration)
        |
  MCP Gateway (mTLS :8443) -> MCP Server -> app01
        |
  Firestore state + BigQuery evidence
```

## Stack

| Area | Choices |
| --- | --- |
| Infrastructure | Terraform, GCP VPC, GKE Autopilot private cluster, Private Google Access |
| Security and identity | Dataplane V2 (Cilium eBPF), Workload Identity, Cloud KMS, Secret Manager, NetworkPolicies |
| Registry and runtime | Artifact Registry, Docker, Kubernetes |
| Application | Python 3.11, FastAPI, FastMCP, Jinja2, OPA Rego |

### Namespaces

`ai-agents`, `ai-governance`, `mcp-gateway`, `mcp`, `app01`, `security`, `shared-services`, `falco`

## Repository layout

Directories exist. Source, manifests, and Terraform modules are added as each piece is built.

```text
terraform/
  _shared/                shared Terraform inputs and conventions
  01-foundation/          VPC, GKE Autopilot, identity, Private Google Access
  02-messaging/           Pub/Sub topics and dead-letter queues
  03-security-sources/    Falco, Trivy, and Prowler infrastructure
  05-remediation/         KMS, approval, and execution-plane infrastructure
src/                      agent and service code
  event-aggregator/ observer/ correlation/ ir-analyst/
  governance/policies/    OPA Rego
  approval/ remediation/ reporting/
  mcp-gateway/ mcp-server/
  broken-app/             app01 workload used as the remediation target
  shared/
docker/                   image build context per service (mirrors src/)
k8s-manifests/
  01-foundation/
  02-security-sources/    falco/ trivy/ prowler/
  03-agents/              one directory per agent
  04-remediation/         mcp-gateway/ mcp-server/
  05-app/app01/
schemas/                  JSON schemas, including the IR Analyst response contract
docs/architecture/        design notes
docs/runbooks/            operator procedures
scripts/
.github/
```

## Status

Scaffold only. No Terraform, application code, manifests, or schemas have been written yet.
