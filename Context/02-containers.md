# Level 2: Container Architecture

## Overview

This document describes the major containers (applications and data stores) that make up the Ephemeral GitHub Actions Runner Platform. Each container represents a separately deployable/runnable unit.

## Container Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     GITHUB INSTANCES                                │
│                                                                     │
│  ┌──────────────────────────┐    ┌──────────────────────────┐     │
│  │ GitHub Enterprise Server │    │    GitHub Cloud          │     │
│  │                          │    │                          │     │
│  │ • Workflows              │    │ • Workflows              │     │
│  │ • Runner groups          │    │ • Runner groups          │     │
│  │ • Listener-based scaling │    │ • Listener-based scaling │     │
│  └──────────────────────────┘    └──────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    AWS EKS (BLUE/GREEN)                             │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Runner Controllers                                           │  │
│  │ • Listener per team                                          │  │
│  │ • Autoscale runners + nodes                                  │  │
│  │ • GitHub Actions Runner Scale Sets                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Worker Nodes                                                 │  │
│  │ • Karpenter autoscaling                                      │  │
│  │ • Scale EC2 instances                                        │  │
│  │ • Spot + On-Demand                                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Ephemeral Runner Pods                                        │  │
│  │ • Runner scale sets                                          │  │
│  │ • High availability                                          │  │
│  │ • Auto-cleanup                                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Node Pools                                                   │  │
│  │ • Warm start pool                                            │  │
│  │ • Isolated high-resource pools                               │  │
│  │ • Custom compute                                             │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Cilium eBPF (CNI)                                            │  │
│  │ • Network policies                                           │  │
│  │ • Encryption (WireGuard)                                     │  │
│  │ • L7 observability (Hubble)                                  │  │
│  │ • High performance                                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┴─────────────────┐
            ▼                                   ▼
┌────────────────────────┐        ┌────────────────────────┐
│ Custom Image Bakery    │        │   OIDC + Vault         │
│                        │        │                        │
│ • Base images          │        │ • GitHub → AWS roles   │
│ • Team custom images   │        │ • GitHub → Vault       │
│ • Security scanning    │        │ • No static creds      │
│ • Automated builds     │        │ • Short-lived tokens   │
└────────────────────────┘        └────────────────────────┘
```

## Containers

### GitHub Enterprise Server
**Technology**: GitHub Enterprise Server (on-premises)

**Responsibilities**:
- Host source code repositories
- Execute workflow orchestration
- Manage runner groups and labels
- Emit webhook events for job queue changes

**Key Interactions**:
- Sends webhook events to Runner Controllers
- Receives runner registration from ephemeral pods
- Routes jobs to appropriate runner groups

### GitHub Cloud
**Technology**: GitHub SaaS (github.com)

**Responsibilities**:
- Host source code repositories (cloud)
- Execute workflow orchestration
- Manage runner groups and labels
- Emit webhook events for job queue changes

**Key Interactions**:
- Sends webhook events to Runner Controllers (separate from Enterprise)
- Receives runner registration from ephemeral pods
- Routes jobs to appropriate runner groups

**Note**: GitHub Cloud and Enterprise Server use completely separate compute pools to ensure isolation.

---

## AWS EKS Cluster (Blue/Green Deployment)

### Runner Controllers
**Technology**: Kubernetes Deployment running GitHub Actions Runner Scale Set Controllers

**Responsibilities**:
- Monitor GitHub webhook queue via listeners (one per team)
- Calculate desired runner count based on pending jobs
- Create/delete ephemeral runner pods
- Manage runner lifecycle (registration, job execution, cleanup)
- Enforce team isolation and resource limits

**Scaling**:
- One listener per team for independent scaling
- Horizontal pod autoscaling based on webhook queue depth
- Triggers Karpenter for node provisioning when needed

**Key Interactions**:
- Consumes webhooks from GitHub (Enterprise + Cloud)
- Creates runner pods on worker nodes
- Coordinates with Karpenter for node scaling

### Worker Nodes
**Technology**: AWS EC2 instances managed by Karpenter

**Responsibilities**:
- Provide compute resources for runner pods
- Auto-scale based on pod scheduling needs
- Support multiple instance types (general, high-CPU, high-memory, GPU)
- Optimize for cost with Spot instances (75% usage)

**Node Types**:
- **Shared Pool**: General-purpose compute for most teams
- **Isolated Pools**: Dedicated nodes for high-resource teams
- **Custom Compute**: GPU, NVMe, high-memory for specialized workloads

**Scaling**:
- Karpenter monitors unschedulable pods
- Provisions right-sized instances within seconds
- Consolidates underutilized nodes for cost efficiency

### Ephemeral Runner Pods
**Technology**: Kubernetes Pods running GitHub Actions Runner

**Responsibilities**:
- Register with GitHub as a self-hosted runner
- Pull and execute a single workflow job
- Provide isolated execution environment
- Cleanup and terminate after job completion

**Lifecycle**:
1. Pod created by Runner Controller
2. Runner registers with GitHub
3. Job assigned and executed
4. Runner de-registers
5. Pod terminated automatically

**Configuration**:
- Uses Custom Image Bakery images
- Mounts secrets from Vault via OIDC
- Assumes AWS IAM roles via OIDC
- Enforces max pods per node for fair I/O

### Node Pools
**Technology**: Kubernetes node labels and taints

**Responsibilities**:
- **Warm Start Pool**: Pre-provisioned nodes with common images for instant job start (~5s)
- **Isolated Pools**: Dedicated compute for high-resource teams (ML, video processing)
- **Custom Compute**: GPU, NVMe, high-memory for specialized workloads

**Benefits**:
- Zero cold start time for common workloads
- Performance isolation (no noisy neighbors)
- Custom hardware support
- Predictable costs per team

### Cilium eBPF (CNI)
**Technology**: Cilium Container Network Interface with eBPF

**Responsibilities**:
- Provide Kubernetes networking (pod-to-pod, pod-to-service)
- Enforce network policies for team isolation
- Encrypt traffic in transit (WireGuard)
- Provide L7 observability via Hubble
- High-performance packet processing via eBPF

**Why Cilium**:
- Network policies for security and isolation
- Transparent encryption without service mesh overhead
- Deep observability for troubleshooting
- Superior performance via kernel bypass (eBPF)

---

## Supporting Containers

### Custom Image Bakery
**Technology**: Automated container image build and scan pipeline

**Responsibilities**:
- Build and maintain base runner images
- Support team-specific custom images
- Automated security scanning (Trivy, Grype)
- Scheduled rebuilds for security patches
- Distribute images to ECR

**Image Types**:
- **Base Images**: Hardened OS with common tools, security patches
- **Custom Images**: Team-specific toolchains, languages, dependencies
- **Specialized Images**: GPU drivers, ML frameworks, video processing

**Security**:
- CVE scanning on every build
- Fail builds with HIGH/CRITICAL vulnerabilities
- Automated patching pipeline
- Immutable image tags

### OIDC + Vault
**Technology**: GitHub OIDC + HashiCorp Vault

**Responsibilities**:
- Authenticate runners to AWS (no static credentials)
- Authenticate runners to Vault (dynamic secrets)
- Issue short-lived tokens (15-60 minutes)
- Automatic credential rotation and revocation
- Full audit trail

**Trust Chain**:
1. Runner pod receives GitHub OIDC token
2. Token contains claims (repo, workflow, team)
3. AWS STS validates token → assumes IAM role
4. Vault validates token → issues dynamic secrets

**Benefits**:
- Zero static credentials stored
- Automatic credential lifecycle
- Per-job credential scoping
- Reduced blast radius

---

## Key Features

### Blue/Green High Availability
- **Active Cluster**: Serves production traffic
- **Standby Cluster**: Ready for instant failover
- **Benefits**: Zero-downtime updates, instant failover, safe testing environment

### Listener-Based Autoscaling
- **Per-Team Listeners**: Each team gets dedicated webhook monitoring
- **Two-Layer Scaling**: Runner Scale Sets (pods) + Karpenter (nodes)
- **Benefits**: Independent team scaling, no cross-team interference

### Developer Experience
- **Warm Start Pools**: ~5 second job start time
- **Custom Image Bakery**: Team-specific tooling
- **Community + SIGs**: Developer community groups
- **Self-Service**: Teams configure their own runners

## Technology Stack

- **Orchestration**: Kubernetes (AWS EKS)
- **Networking**: Cilium eBPF
- **Compute**: AWS EC2 (Karpenter autoscaling)
- **Runners**: GitHub Actions Runner Scale Sets
- **Secrets**: HashiCorp Vault
- **Auth**: GitHub OIDC + AWS STS
- **Observability**: Hubble, CloudWatch, Prometheus

## Deployment Model

**Blue/Green Strategy**:
- Two identical EKS clusters (Blue and Green)
- Active cluster handles production traffic
- Standby cluster ready for failover
- Updates deployed to standby, then traffic switched
- Zero downtime, instant rollback capability