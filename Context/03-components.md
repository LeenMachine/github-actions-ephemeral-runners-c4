# Level 3: Component Diagram

## Overview

This document provides a detailed view of the internal components within the Runner Controller container. These components work together to manage the lifecycle of ephemeral GitHub Actions runners.

## Component Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    RUNNER CONTROLLER                            │
│                                                                 │
│  ┌──────────────────────┐      ┌──────────────────────┐        │
│  │  Webhook Router      │      │  Pod Provisioner     │        │
│  │                      │      │                      │        │
│  │ • Routes Enterprise/ │      │ • Creates pods       │        │
│  │   Cloud webhooks     │      │ • Enforces max per   │        │
│  │ • Extracts runner    │      │   node               │        │
│  │   groups             │      │ • Manages lifecycle  │        │
│  │ • Validates payloads │      │                      │        │
│  └──────────┬───────────┘      └──────────▲───────────┘        │
│             │                             │                     │
│             ▼                             │                     │
│  ┌──────────────────────┐      ┌──────────┴───────────┐        │
│  │  Team Verifier       │      │  OIDC Manager        │        │
│  │                      │      │                      │        │
│  │ • GitHub Teams API   │      │ • AWS IAM role       │        │
│  │ • Caches membership  │      │   mapping            │        │
│  │ • Enforces team      │      │ • Vault auth         │        │
│  │   isolation          │      │ • Token management   │        │
│  └──────────────────────┘      └──────────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  Autoscaling Flow     │
              │                       │
              │  GitHub Webhook       │
              │         ↓             │
              │  Listener (per team)  │
              │         ↓             │
              │  Runner Scale Set     │
              │         ↓             │
              │  Pod Creation         │
              │         ↓             │
              │  Karpenter            │
              │         ↓             │
              │  EC2 Node Provision   │
              └───────────────────────┘
```

## Core Components

### Webhook Router

**Purpose**: Route incoming GitHub webhooks to the appropriate runner controllers

**Responsibilities**:
- Receive webhooks from GitHub Enterprise Server and GitHub Cloud
- Parse webhook payloads to extract job metadata
- Extract runner group and label information
- Route webhooks to the correct listener instance
- Validate webhook signatures for security

**Technology**: Go-based HTTP server with GitHub webhook validation

**Key Logic**:
```
Incoming Webhook
  → Verify signature
  → Parse payload
  → Extract runner_group
  → Extract labels
  → Route to appropriate listener
```

**Challenges Solved**:
- Multi-instance GitHub support (Enterprise vs Cloud)
- Runner group isolation
- Webhook authentication
- High-throughput processing (1000s of webhooks/minute)

---

### Team Verifier

**Purpose**: Enforce team-based isolation and access control

**Responsibilities**:
- Integrate with GitHub Teams API
- Cache team membership data (5 minute TTL)
- Validate job requester is member of allowed teams
- Enforce runner group → team mapping
- Audit team access attempts

**Technology**: GitHub API client with Redis caching layer

**Verification Flow**:
```
Job Request
  → Extract requester username
  → Check cache for team membership
  → If miss: Query GitHub Teams API
  → Validate requester in allowed teams
  → Allow or deny job assignment
```

**Benefits**:
- Prevents cross-team job execution
- Audit trail of who ran what
- Automatic team sync from GitHub
- Reduced GitHub API calls via caching

---

### Pod Provisioner

**Purpose**: Create and manage ephemeral runner pods

**Responsibilities**:
- Create runner pods based on scale set demands
- Enforce max pods per node (fair I/O allocation)
- Apply node selectors and affinity rules
- Configure pod resources (CPU, memory, storage)
- Attach appropriate service accounts
- Inject environment variables and secrets
- Manage pod lifecycle (create, monitor, cleanup)

**Technology**: Kubernetes client-go library

**Pod Configuration**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: runner-<unique-id>
  labels:
    app: github-runner
    team: <team-name>
    pool: <node-pool>
spec:
  serviceAccountName: github-runner-sa
  nodeSelector:
    pool: <warm-start|isolated|custom>
  affinity:
    podAntiAffinity: # Max pods per node
  containers:
  - name: runner
    image: <custom-image>
    env:
    - name: GITHUB_TOKEN
      valueFrom:
        secretKeyRef: ...
```

**Key Features**:
- **Fair I/O**: Max 10 pods per node prevents noisy neighbors
- **Node Pool Selection**: Routes pods to appropriate node pools
- **Automatic Cleanup**: Pods terminate after job completion
- **Resource Limits**: Prevents resource exhaustion

---

### OIDC Manager

**Purpose**: Manage OpenID Connect authentication flows

**Responsibilities**:
- Configure AWS IAM OIDC trust relationship
- Map runner pods to IAM roles based on team/repo
- Configure Vault OIDC authentication
- Issue and refresh short-lived tokens
- Audit credential usage

**Technology**: AWS STS, Vault API client

**Trust Flow (AWS)**:
```
1. Pod starts with GitHub OIDC token
2. Token claims include:
   - Repository: org/repo
   - Workflow: .github/workflows/ci.yml
   - Job: build
   - Team: eng-platform
3. AWS STS validates token via GitHub OIDC provider
4. STS assumes IAM role: github-runner-<team>-role
5. Temporary AWS credentials issued (15-60 min)
```

**Trust Flow (Vault)**:
```
1. Pod authenticates to Vault with GitHub OIDC token
2. Vault validates token via GitHub OIDC
3. Vault issues dynamic secrets based on policy
4. Secrets injected into pod environment
5. Secrets auto-revoked after TTL
```

**Benefits**:
- Zero static credentials
- Automatic credential rotation
- Per-job credential scoping
- Full audit trail in CloudTrail and Vault logs

---

## Node Pool Strategy

### Shared Warm Start Pool

**Purpose**: Zero cold start for common workloads

**Characteristics**:
- Pre-provisioned nodes with common images cached
- ~5 second job start time
- General-purpose compute (m5, m6i instances)
- Handles 80% of workloads

**How It Works**:
1. Karpenter maintains minimum node count
2. Nodes pre-pull common images
3. Pods scheduled to warm nodes instantly
4. Karpenter scales out for additional demand

### Isolated High-Resource Pools

**Purpose**: Prevent noisy neighbor problems

**Use Cases**:
- ML model training (high CPU/GPU)
- Video processing (high I/O)
- Large monorepo builds (high disk I/O)

**Characteristics**:
- Dedicated nodes per high-resource team
- Custom instance types (c6i, r6i, p3)
- Resource guarantees
- Cost allocated per team

**Benefits**:
- Predictable performance
- No cross-team interference
- Custom hardware support

### Custom Compute

**Purpose**: Support specialized workloads

**Examples**:
- **GPU Pools**: p3, g4dn instances for ML/rendering
- **High-Memory Pools**: r6i instances for in-memory processing
- **NVMe Pools**: i3, i4i instances for high-IOPS workloads

**Configuration**:
```yaml
nodeSelector:
  pool: gpu-pool
  nvidia.com/gpu: "true"
resources:
  limits:
    nvidia.com/gpu: 1
```

---

## Security Components

### OIDC to AWS (No Static Credentials)

**Flow**:
```
GitHub Runner Pod
  → Contains OIDC token with claims
  → AWS STS validates token
  → Assumes IAM role: github-runner-<team>-role
  → Returns temporary credentials (15-60 min)
  → Runner uses credentials for AWS API calls
  → Credentials auto-expire
```

**IAM Role Trust Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:org/repo:*"
        }
      }
    }
  ]
}
```

### OIDC to Vault

**Flow**:
```
GitHub Runner Pod
  → Authenticates to Vault with OIDC token
  → Vault validates claims
  → Issues dynamic secrets (DB creds, API keys)
  → Secrets auto-revoke after TTL
```

**Benefits**:
- Database credentials rotated per job
- API keys scoped to specific workflow
- Automatic cleanup
- No credential sprawl

### Cilium Encryption (WireGuard)

**Purpose**: Encrypt pod-to-pod traffic

**How It Works**:
- Cilium installs WireGuard kernel module on nodes
- Automatic key exchange between nodes
- Transparent encryption for all pod traffic
- No application changes required

**Benefits**:
- Defense in depth
- Compliance requirements (data in transit)
- Minimal performance overhead (~5%)

### Team Verification

**Purpose**: Ensure runners only execute jobs for authorized teams

**Enforcement**:
1. Job webhook arrives with requester info
2. Team Verifier checks GitHub Teams API
3. If requester not in allowed team → reject job
4. Audit log records decision

---

## Autoscaling Flow

### End-to-End Scaling

```
1. Developer pushes code to GitHub
   ↓
2. GitHub workflow triggered
   ↓
3. Job enters queue
   ↓
4. GitHub sends webhook to Listener
   ↓
5. Listener calculates desired runner count
   ↓
6. Runner Scale Set creates pod spec
   ↓
7. Kubernetes scheduler places pod
   ↓
8. If no node available → pod Pending
   ↓
9. Karpenter sees pending pod
   ↓
10. Karpenter provisions EC2 instance
    ↓
11. Node joins cluster
    ↓
12. Pod scheduled to node
    ↓
13. Runner starts, registers with GitHub
    ↓
14. Job assigned to runner
    ↓
15. Job executes
    ↓
16. Runner de-registers
    ↓
17. Pod terminates
    ↓
18. Karpenter consolidates unused nodes
```

### Two-Layer Scaling

**Layer 1: Runner Scale Sets (Pods)**
- Scale based on webhook queue depth
- Fast scaling (seconds)
- Creates pod specs

**Layer 2: Karpenter (Nodes)**
- Scale based on pod scheduling needs
- Provisions right-sized EC2 instances
- Consolidates underutilized nodes

**Benefits**:
- Fast response to load spikes
- Efficient resource utilization
- Cost optimization

---

## Observability Components

### Hubble (L7 Network Observability)

**Purpose**: Provide deep network visibility via eBPF

**Capabilities**:
- Real-time network flow monitoring
- DNS query/response tracking
- HTTP/gRPC request tracing
- Network policy violations
- Service dependency mapping

**Use Cases**:
- Debug connectivity issues
- Identify noisy neighbor pods
- Audit network policy effectiveness
- Troubleshoot DNS failures

### Metrics & Monitoring

**CloudWatch Metrics**:
- Runner pod count per team
- Job queue depth
- Node count per pool
- Spot interruption rate
- Cost per team

**Prometheus Metrics**:
- Pod start time (P50, P95, P99)
- Job success/failure rate
- Resource utilization per pod
- Karpenter provisioning time

**Alerts**:
- Pending pods > 5 minutes
- Node provisioning failures
- Spot interruption rate > 10%
- Job failure rate > 5%

---

## Summary

The Runner Controller architecture is designed for:
- **Scalability**: Handles 1000s of parallel jobs
- **Multi-Tenancy**: Team isolation via listeners and node pools
- **Security**: OIDC everywhere, no static credentials
- **Performance**: Warm start pools, two-layer autoscaling
- **Cost Efficiency**: Spot instances, automatic cleanup
- **Observability**: Deep visibility via Hubble and metrics