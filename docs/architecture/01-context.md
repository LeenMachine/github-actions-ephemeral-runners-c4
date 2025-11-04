# Level 1: System Context

## Overview

The Ephemeral GitHub Actions Runner Platform is an enterprise-scale infrastructure that provides on-demand, ephemeral CI/CD runners for GitHub Actions workflows. The platform serves 1000+ developers across the organization, handling thousands of parallel jobs at peak load.

## System Context Diagram

```
┌──────────────┐
│  Developers  │
│              │
│ Push code,   │
│ trigger      │
│ workflows    │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────┐
│   GitHub Instances           │
│                              │
│  ┌────────────────────────┐  │
│  │ GitHub Enterprise      │  │
│  │ Server (On-premises)   │  │
│  └────────────────────────┘  │
│                              │
│  ┌────────────────────────┐  │
│  │ GitHub Cloud (SaaS)    │  │
│  └────────────────────────┘  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│   Ephemeral GitHub Actions Runner Platform  │
│                                              │
│   Multi-instance runner provisioning with   │
│   separate compute for Enterprise and Cloud │
└──────────────┬───────────────────────────────┘
               │
       ┌───────┴───────┐
       ▼               ▼
┌─────────────┐  ┌──────────────────┐
│  AWS Cloud  │  │ HashiCorp Vault  │
│             │  │                  │
│ • EKS       │  │ • Secrets mgmt   │
│ • OIDC      │  │ • OIDC           │
│ • VPC       │  │ • Dynamic creds  │
└─────────────┘  └──────────────────┘
```

## Actors

### Developers
- **Role**: End users of the platform
- **Actions**: 
  - Push code to GitHub repositories
  - Trigger CI/CD workflows
  - Configure workflow files
  - Monitor job execution
- **Count**: 1000+ developers across the organization

## External Systems

### GitHub Enterprise Server
- **Type**: On-premises GitHub instance
- **Purpose**: Version control and workflow orchestration
- **Integration**: Listener-based webhook consumption
- **Runner Groups**: Separate compute pools for different teams

### GitHub Cloud
- **Type**: GitHub SaaS (github.com)
- **Purpose**: Cloud-based version control and workflow orchestration
- **Integration**: Listener-based webhook consumption
- **Runner Groups**: Separate compute pools from Enterprise instance

## The Platform (System of Interest)

### Ephemeral GitHub Actions Runner Platform
- **Purpose**: Provide scalable, ephemeral CI/CD runners
- **Key Characteristics**:
  - Multi-instance support (Enterprise + Cloud)
  - Listener-based autoscaling per team
  - Blue/Green high availability deployment
  - Warm start pools for instant job execution
  - Custom image support per team
  
### Core Capabilities
1. **Dynamic Runner Provisioning**: Create runners on-demand based on workflow queue
2. **Multi-Tenancy**: Isolate teams with dedicated listeners and optional node pools
3. **Auto-Scaling**: Two-layer scaling (runners and nodes) via Scale Sets and Karpenter
4. **Security**: OIDC-based authentication with no static credentials
5. **Cost Optimization**: Ephemeral runners with automatic cleanup

## Supporting Systems

### AWS Cloud
- **EKS Clusters**: Kubernetes orchestration (Blue/Green deployment)
- **OIDC Integration**: Trust relationship for credential-free AWS access
- **VPC**: Expanded 4x for IP address space
- **Compute**: EC2 instances (Spot + On-Demand)

### HashiCorp Vault
- **Purpose**: Dynamic secrets management
- **Integration**: OIDC trust with GitHub
- **Secrets**: Database credentials, API keys, certificates
- **Token Lifecycle**: Short-lived tokens (15-60 minutes)

## Key Relationships

1. **Developers → GitHub**: Developers push code and trigger workflows on either GitHub Enterprise Server or GitHub Cloud
2. **GitHub → Platform**: Separate runner compute for each GitHub instance, routed via runner groups and labels
3. **Platform → AWS**: OIDC trust enables credential-free access to AWS resources
4. **Platform → Vault**: OIDC trust enables dynamic secret injection into runner pods
5. **Multi-Instance Isolation**: Enterprise and Cloud workloads run on separate compute with independent scaling

## Scale & Performance

- **Throughput**: Handles 1000s of parallel jobs at peak
- **Users**: Serves 1000+ developers across organization
- **Availability**: 99.9% uptime SLA
- **Start Time**: ~5 seconds (warm start pool)
- **Cost**: 70% savings vs persistent runners

## Security Posture

- **Zero Static Credentials**: All authentication via OIDC short-lived tokens
- **Network Isolation**: Cilium eBPF network policies
- **Encryption**: WireGuard for data in transit
- **Team Verification**: GitHub Teams API integration
- **Image Security**: Automated scanning in Custom Image Bakery

## Design Principles

1. **Developer Experience First**: Optimize for what developers love
2. **Ephemeral by Default**: No persistent state, auto-cleanup
3. **High Availability**: Blue/Green deployment for zero downtime
4. **Security by Design**: OIDC everywhere, no static credentials
5. **Cost Efficiency**: Pay only for what you use

## Out of Scope

- GitHub Enterprise Server infrastructure itself
- GitHub Cloud SaaS platform
- Developer workstations and local environments
- Source code repository management
- Artifact storage (handled by separate systems)