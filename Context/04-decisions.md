# Architecture Decisions

## Overview

This document captures the key architectural decisions made during the design and implementation of the Ephemeral GitHub Actions Runner Platform. Each decision includes the problem context, solution chosen, alternatives considered, and the rationale.

---

## ADR-001: Blue/Green Deployment Strategy

### Problem
Need zero-downtime updates and instant disaster recovery for a mission-critical CI/CD platform serving 1000+ developers.

### Solution
Maintain two identical EKS clusters (Blue and Green) with one active and one standby at all times.

### Architecture
- **Active Cluster**: Handles all production traffic
- **Standby Cluster**: Fully provisioned and ready for failover
- **Traffic Switching**: DNS/load balancer cutover between clusters
- **Update Process**: Deploy to standby → validate → switch traffic → old cluster becomes standby

### Benefits
- **Zero Downtime**: Switch between clusters in seconds
- **Instant Failover**: Standby cluster ready for disaster recovery
- **Safe Testing**: Test updates on standby before production
- **Quick Rollback**: Switch back to previous cluster if issues arise
- **Disaster Recovery**: Multi-AZ protection

### Trade-offs
- **Cost**: Running two clusters doubles infrastructure cost
  - Mitigation: Standby runs at reduced capacity until needed
- **Complexity**: Must keep clusters in sync
  - Mitigation: Infrastructure as Code (Terraform) ensures consistency

### Status
✅ Implemented and operational

---

## ADR-002: Listener-Based Autoscaling

### Problem
- Need to support multiple GitHub instances (Enterprise Server + Cloud)
- Teams have vastly different scaling needs
- Avoid cross-team interference during load spikes
- Traditional polling-based scaling has delays and API rate limits

### Solution
Implement listener-based autoscaling with one listener per team

### Architecture
- **GitHub Webhook Stream**: Real-time job queue updates
- **Listener per Team**: Dedicated listener monitors team's queue
- **Independent Scaling**: Each team scales runners independently
- **Two-Layer Autoscaling**: Runner Scale Sets (pods) + Karpenter (nodes)

### Benefits
- **Real-Time**: Instant response to job queue changes (vs polling delays)
- **Team Isolation**: One team's load spike doesn't affect others
- **Multi-Instance**: Separate listeners for Enterprise and Cloud
- **API Efficient**: Webhooks instead of constant polling
- **Fine-Grained Control**: Per-team scaling parameters

### Alternatives Considered
1. **Single Shared Listener**
   - ❌ Cross-team interference
   - ❌ No team-specific scaling policies
   
2. **Polling-Based**
   - ❌ Delays (30-60 second polling interval)
   - ❌ GitHub API rate limits
   - ❌ Higher latency

### Status
✅ Implemented and operational

---

## ADR-003: Warm Start Pool Strategy

### Problem
Cold start adds 30-60 seconds of latency per job due to:
- Node provisioning (20-30s)
- Image pulling (10-30s)
- Pod initialization (5-10s)

This poor developer experience was the #1 complaint.

### Solution
Maintain pre-warmed runner pool with most common images

### Implementation
- **Minimum Node Count**: Karpenter maintains 10-20 nodes warm
- **Image Pre-Pulling**: DaemonSet pre-pulls common images to all nodes
- **Fast Path**: Pods scheduled to warm nodes start in ~5 seconds
- **Overflow Scaling**: Cold nodes provisioned when warm pool exhausted

### Results
- **Start Time**: Reduced from 45s average to ~5s (89% improvement)
- **Developer Satisfaction**: 40% increase in platform satisfaction scores
- **Cost**: Minimal (~$500/month for warm pool)

### Trade-offs
- **Idle Cost**: Warm nodes cost money even when unused
  - Mitigation: Right-size warm pool based on usage patterns
  - Mitigation: Auto-scale warm pool during off-hours
- **Image Storage**: Pre-pulled images consume disk space
  - Mitigation: Limit to 5-10 most common images

### Status
✅ Implemented and operational

---

## ADR-004: Node Pool Isolation

### Problem
High-resource teams (ML, video processing, large builds) create "noisy neighbor" problems:
- Disk I/O saturation affects other teams
- CPU/memory exhaustion causes pod evictions
- Unpredictable performance

### Solution
Provide dedicated node pools for high-resource teams

### Architecture
- **Shared Pool**: General compute for most teams (80% of workloads)
- **Isolated Pools**: Dedicated nodes per high-resource team
- **Custom Compute**: GPU, high-memory, NVMe for specialized needs

### Implementation
```yaml
# High-resource team pod spec
nodeSelector:
  pool: ml-team-isolated
resources:
  requests:
    cpu: "8"
    memory: "32Gi"
  limits:
    cpu: "16"
    memory: "64Gi"
```

### Benefits
- **Performance Isolation**: Teams don't interfere with each other
- **Custom Hardware**: GPU, NVMe, high-memory support
- **Predictable Costs**: Resource usage attributable per team
- **SLA Guarantees**: High-resource teams get dedicated capacity

### Results
- 60% reduction in "noisy neighbor" incidents
- 95% reduction in support tickets related to slow builds
- ML team build times reduced from 45min to 15min

### Trade-offs
- **Resource Efficiency**: Some node pools may be underutilized
  - Mitigation: Right-size pools based on actual usage
  - Mitigation: Use Cluster Autoscaler for isolated pools too

### Status
✅ Implemented for 5 high-resource teams

---

## ADR-005: IP Address Space Expansion

### Problem
- Original VPC: /20 CIDR (4,096 IPs)
- At scale: 1000s of parallel pods
- IP exhaustion became regular occurrence
- Job failures due to "no IPs available"

### Solution
Expand VPC to /18 CIDR (16,384 IPs) - 4x increase

### Implementation
1. Created new subnets with /18 CIDR
2. Migrated workloads to new subnets
3. Deprecated old /20 subnets
4. Updated route tables and security groups

### Results
- **Capacity**: 4x IP address space
- **Headroom**: Supports 4x growth without re-architecture
- **Reliability**: Zero IP exhaustion incidents post-expansion

### Alternatives Considered
1. **IPv6**
   - ❌ GitHub Actions doesn't fully support IPv6
   - ❌ Many internal services IPv4-only
   
2. **Multiple VPCs**
   - ❌ Complex cross-VPC routing
   - ❌ Operational overhead

### Status
✅ Completed

---

## ADR-006: Cilium eBPF for Networking

### Problem
Traditional Kubernetes networking (kube-proxy, iptables) has limitations:
- Performance bottleneck at scale
- Limited network observability
- No native encryption
- Difficult troubleshooting

### Solution
Deploy Cilium as the CNI with eBPF datapath

### Benefits
- **Performance**: eBPF bypasses netfilter/iptables overhead
  - 3x throughput improvement in benchmarks
- **Network Policies**: Enforce team isolation at L3/L4/L7
- **Encryption**: WireGuard encryption in transit (transparent to apps)
- **Observability**: Hubble provides deep L7 visibility
- **Modern**: Actively developed, CNCF graduated project

### Implementation
```yaml
# Network policy example
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: team-isolation
spec:
  endpointSelector:
    matchLabels:
      team: ml-team
  ingress:
  - fromEndpoints:
    - matchLabels:
        team: ml-team
```

### Results
- **Performance**: 3x network throughput vs iptables
- **Security**: WireGuard encryption enabled with <5% overhead
- **Observability**: Hubble reduced troubleshooting time by 70%
- **Policies**: 100+ network policies enforcing team isolation

### Alternatives Considered
1. **Calico**
   - ✅ Mature, widely used
   - ❌ No eBPF datapath
   - ❌ Limited L7 visibility
   
2. **AWS VPC CNI**
   - ✅ Native AWS integration
   - ❌ Consumes VPC IPs per pod (wasteful)
   - ❌ No L7 observability

### Status
✅ Implemented and operational

---

## ADR-007: OIDC + Vault Architecture

### Problem
Traditional approach stores static credentials:
- AWS access keys in GitHub Secrets
- Database passwords in Kubernetes secrets
- API keys hardcoded in runner images

**Risks**:
- Long-lived credentials
- Difficult rotation
- Hard to audit
- High blast radius if compromised

### Solution
Zero-trust credential framework using OIDC

### Architecture

**GitHub OIDC → AWS IAM**
```
Runner Pod
  → Contains GitHub OIDC token (JWT)
  → Token claims: repo, workflow, job
  → AWS STS validates token
  → Assumes IAM role
  → Returns temporary credentials (15-60 min)
```

**GitHub OIDC → Vault**
```
Runner Pod
  → Authenticates to Vault with OIDC token
  → Vault validates claims
  → Issues dynamic secrets (DB creds, API keys)
  → Auto-revoked after TTL
```

### Benefits
- **No Static Credentials**: Zero long-lived secrets stored
- **Short-Lived Tokens**: 15-60 minute TTL
- **Automatic Rotation**: Credentials auto-rotate per job
- **Fine-Grained Permissions**: IAM/Vault policies per team/repo
- **Full Audit Trail**: CloudTrail and Vault logs every access
- **Reduced Blast Radius**: Compromised token limited to single job

### Results
- 100% elimination of static AWS credentials
- 95% reduction in credential rotation toil
- Zero credential leaks in 18 months of operation
- Full SOC2 compliance for credential management

### Alternatives Considered
1. **Static Credentials in Secrets**
   - ❌ Long-lived secrets
   - ❌ Manual rotation
   - ❌ High security risk
   
2. **IAM Roles for Service Accounts (IRSA)**
   - ✅ Works for AWS
   - ❌ Doesn't support Vault or other systems
   - ❌ Less flexible than OIDC

### Status
✅ Implemented for AWS and Vault

---

## ADR-008: Custom Image Bakery

### Problem
- Teams need specialized tooling (language runtimes, build tools, ML frameworks)
- Manual image builds are inconsistent and insecure
- No automated security scanning
- Image sprawl (100+ unmanaged images)

### Solution
Centralized Custom Image Bakery with automated pipeline

### Architecture
```
1. Base Images
   - Ubuntu 22.04 LTS hardened
   - Common tools (git, docker, curl, etc)
   - Security patches applied
   
2. Team Custom Images
   - Dockerfile in team repo
   - Automated builds via CI/CD
   - Inherits from base image
   
3. Security Scanning
   - Trivy CVE scanning
   - Grype vulnerability checks
   - Fail builds with HIGH/CRITICAL
   
4. Distribution
   - Push to ECR
   - Immutable tags
   - Image pull policy: Always
```

### Benefits
- **Consistency**: All images built from same base
- **Security**: Automated CVE scanning, patch pipeline
- **Self-Service**: Teams build custom images themselves
- **Governance**: Central control over base image
- **Compliance**: Audit trail of all image builds

### Results
- Reduced image count from 100+ to 25 managed images
- 95% reduction in security vulnerabilities
- Zero manual image builds
- 40% faster image pulls (via ECR)

### Alternatives Considered
1. **Pre-Built GitHub Images**
   - ❌ No customization
   - ❌ Slow to update
   - ❌ Limited tool selection
   
2. **Team-Managed Images**
   - ❌ No security scanning
   - ❌ Inconsistent quality
   - ❌ Image sprawl

### Status
✅ Implemented with 25 images built

---

## ADR-009: Fair I/O Allocation (Max Pods Per Node)

### Problem
- Disk I/O saturation when 50+ pods on single node
- Build cache thrashing
- Poor performance for all tenants on node

### Solution
Enforce maximum 10 pods per node via pod anti-affinity

### Implementation
```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: github-runner
      topologyKey: kubernetes.io/hostname
      maxPodsPerNode: 10
```

### Results
- 90% reduction in I/O saturation incidents
- More predictable build times
- Better disk cache effectiveness

### Trade-offs
- **Resource Efficiency**: Some CPU/memory underutilized
  - Mitigation: Accept trade-off for performance predictability
- **More Nodes Needed**: Lower pod density requires more nodes
  - Mitigation: Offset by Spot instance cost savings

### Status
✅ Implemented globally

---

## ADR-010: GitHub Actions Runner Scale Sets

### Problem
Previous solution (custom runner controller) required:
- Custom code maintenance
- GitHub API integration complexity
- Scaling logic development

### Solution
Adopt GitHub Actions Runner Scale Sets (official solution)

### Benefits
- **Official Support**: Maintained by GitHub
- **Webhook Integration**: Built-in listener support
- **Automatic Updates**: GitHub handles controller updates
- **Best Practices**: Implements GitHub's recommended patterns
- **Community**: Large user base, active development

### Migration
1. Deployed Runner Scale Sets alongside custom controller
2. Migrated teams incrementally
3. Validated feature parity
4. Decommissioned custom controller

### Results
- 80% reduction in controller maintenance
- Zero GitHub API integration bugs
- Automatic feature updates from GitHub
- Better documentation and community support

### Status
✅ Migrated and operational

---

## Summary of Key Decisions

| Decision | Problem | Solution | Status |
|----------|---------|----------|--------|
| Blue/Green | Zero downtime | Dual clusters | ✅ Implemented |
| Listener Autoscaling | Team isolation | Per-team listeners | ✅ Implemented |
| Warm Start | Cold start latency | Pre-warmed nodes | ✅ Implemented |
| Node Pool Isolation | Noisy neighbors | Dedicated pools | ✅ Implemented |
| IP Expansion | IP exhaustion | 4x VPC size | ✅ Completed |
| Cilium eBPF | Performance, observability | eBPF CNI | ✅ Implemented |
| OIDC + Vault | Static credentials | Zero-trust OIDC | ✅ Implemented |
| Image Bakery | Image sprawl | Centralized builds | ✅ Implemented |
| Fair I/O | Disk saturation | Max pods per node | ✅ Implemented |
| Runner Scale Sets | Maintenance burden | Official GitHub solution | ✅ Migrated |

---

## Lessons Learned

### What Worked Well
1. **Incremental Rollout**: Blue/Green enabled safe, gradual changes
2. **Developer Feedback**: Monthly showcases identified issues early
3. **Observability First**: Hubble saved countless hours of debugging
4. **Security by Design**: OIDC prevented credential incidents

### What We'd Do Differently
1. **Start with Larger VPC**: IP expansion was painful migration
2. **Node Pool Strategy Earlier**: Noisy neighbors caused early pain
3. **More Aggressive Warm Pool**: Could have been even faster

### Future Considerations
1. **ARM64 Nodes**: Graviton instances for cost savings
2. **Fargate Support**: For specific low-throughput workloads
3. **Multi-Region**: Disaster recovery across AWS regions
4. **GitLab Support**: Expand beyond GitHub