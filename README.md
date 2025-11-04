## Ephemeral GitHub Actions Runners - C4 Architecture

## Repository Structure

```
ephemeral-github-runners/
├── README.md                          # This file
├── docs/
│   ├── architecture/
│   │   ├── 01-context.md             # Level 1: System Context
│   │   ├── 02-containers.md          # Level 2: Containers
│   │   ├── 03-components.md          # Level 3: Components
│   │   └── 04-decisions.md           # Architecture Decision Records
│   ├── team-topologies.md            # Team structure and culture
│   └── deployment-strategy.md        # Blue/Green deployment approach
├── diagrams/
│   ├── c4-interactive.html           # Interactive C4 diagram
│   └── images/                       # Static diagram exports
└── CONTRIBUTING.md                   # How to contribute
```

## Overview

This repository documents the architecture of an enterprise-scale ephemeral GitHub Actions runner platform that serves thousands of developers across an organization.

### Key Features

- **Multi-Instance Support**: Separate compute for GitHub Enterprise Server and GitHub Cloud
- **Auto-Scaling**: Listener-based per-team scaling with Karpenter node provisioning
- **Security**: OIDC trust with AWS and HashiCorp Vault, Cilium eBPF network policies
- **High Availability**: Blue/Green deployment strategy with active/standby clusters
- **Developer Experience**: Built with Team Topologies principles, psychological safety, and Kanban flow
- **Scale**: Handles 1000s of parallel jobs serving entire engineering organization

### Architecture Highlights

- **Warm Start Pools**: Zero cold start for common workloads (~5 second start time)
- **Isolated Node Pools**: Dedicated compute for high-resource teams
- **Custom Image Bakery**: Team-specific runner images with security scanning
- **Fair I/O Allocation**: Max pods per node prevents noisy neighbors
- **IP Expansion**: VPC scaled 4x to support growth

## Team Topologies & Human Engineering

This platform was built with a **people-first approach**:

- **Psychological Safety**: Created environment where teams could experiment and learn
- **Developer Experience > Features**: Optimized for what developers loved, not just feature delivery
- **Kanban Flow**: Managed changing requirements while maintaining delivery momentum
- **Community Engineering**: Built SIG groups, showcases, and developer community
- **Voice of Customer**: Internal developers were priority #1 in all decisions

The team used principles from *Team Topologies* to structure work around stream-aligned teams, enabling teams, and platform teams.

## Quick Start

### View the Interactive Diagram

Open `diagrams/c4-interactive.html` in any modern web browser to explore the architecture interactively.

### Navigate the Documentation

1. Start with [System Context](docs/architecture/01-context.md) - understand the system landscape
2. Review [Container Architecture](docs/architecture/02-containers.md) - see the major components
3. Deep dive into [Components](docs/architecture/03-components.md) - understand internal structure
4. Read [Architecture Decisions](docs/architecture/04-decisions.md) - learn why choices were made
5. Explore [Team Topologies](docs/team-topologies.md) - understand the people and culture aspects
6. Review [Deployment Strategy](docs/deployment-strategy.md) - Blue/Green HA approach

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on how to contribute to this documentation.

## License

For Demonstration Purposes Only - not for public distribution.

---

## Architecture at a Glance

### System Context (Level 1)
```
Developers → GitHub (Enterprise + Cloud) → Runner Platform (Blue/Green) → AWS/Vault
```

### Key Technologies
- **Container Orchestration**: Kubernetes (AWS EKS)
- **Networking**: Cilium eBPF (CNI)
- **Secrets**: HashiCorp Vault + GitHub OIDC
- **Autoscaling**: GitHub Actions Runner Scale Sets + Karpenter
- **High Availability**: Blue/Green cluster deployment
- **Observability**: Hubble, CloudWatch, Prometheus

### Scale Metrics
- **1000+ developers** served across the organization
- **1000s of parallel jobs** at peak load
- **500+ concurrent runners** typical
- **~5 second** warm start time
- **70% cost savings** vs persistent runners
- **99.9% uptime** SLA achieved