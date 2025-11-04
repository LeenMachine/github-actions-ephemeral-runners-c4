# Blue/Green Deployment Strategy

## Overview

The Ephemeral GitHub Actions Runner Platform uses a Blue/Green deployment strategy to achieve zero-downtime updates, instant disaster recovery, and safe testing of infrastructure changes. This document details the architecture, processes, and operational procedures.

---

## What is Blue/Green Deployment?

Blue/Green deployment is a release strategy that maintains two identical production environments:

- **Blue Environment**: Currently serving production traffic (active)
- **Green Environment**: Identical to Blue but on standby (passive)

During updates:
1. Deploy changes to the Green (standby) environment
2. Validate Green environment thoroughly
3. Switch traffic from Blue to Green
4. Blue becomes the new standby environment

---

## Architecture

### Two Identical EKS Clusters

```
┌─────────────────────────────────────────────────────────────────┐
│                      ROUTE 53 / LOAD BALANCER                   │
│                                                                 │
│              Traffic routing controlled by DNS/LB               │
└────────────┬───────────────────────────────────┬────────────────┘
             │                                   │
             ▼                                   ▼
┌────────────────────────┐         ┌────────────────────────┐
│   BLUE CLUSTER         │         │   GREEN CLUSTER        │
│   (ACTIVE)             │         │   (STANDBY)            │
│                        │         │                        │
│ ✓ Runner Controllers   │         │ ✓ Runner Controllers   │
│ ✓ Worker Nodes         │         │ ✓ Worker Nodes         │
│ ✓ Karpenter            │         │ ✓ Karpenter            │
│ ✓ Cilium               │         │ ✓ Cilium               │
│ ✓ Observability        │         │ ✓ Observability        │
│                        │         │                        │
│ Serving 100% traffic   │         │ Ready for failover     │
└────────────────────────┘         └────────────────────────┘
```

### Cluster Configuration

Both clusters are **identical** in configuration:
- Same Kubernetes version
- Same node instance types
- Same Karpenter provisioners
- Same runner controller configuration
- Same network policies (Cilium)
- Same observability stack

**Differences**:
- Cluster names (blue vs green)
- VPC subnets (isolated)
- Standby cluster may run at reduced scale when not active

---

## Traffic Routing

### DNS-Based Routing

**Primary Method**: Route 53 weighted routing

```
runners.company.com
  ├─ Blue Cluster LB (Weight: 100)
  └─ Green Cluster LB (Weight: 0)

# During switchover:
runners.company.com
  ├─ Blue Cluster LB (Weight: 0)
  └─ Green Cluster LB (Weight: 100)
```

**Advantages**:
- Simple DNS update
- TTL controls propagation speed
- Easy rollback (flip weights)

**TTL Configuration**:
- Normal: 300 seconds (5 minutes)
- During deployment: 60 seconds (faster switchover)

### Load Balancer-Based Routing

**Alternative Method**: Application Load Balancer with target groups

```
ALB Listener Rules
  ├─ Blue Target Group (Active)
  └─ Green Target Group (Standby)
```

**Used For**:
- GitHub webhook receivers (requires instant cutover)
- API endpoints (avoid DNS caching issues)

---

## Deployment Process

### 1. Preparation Phase

**Verify Current State**:
```bash
# Identify active cluster
kubectl config use-context blue-cluster
kubectl get nodes
kubectl get deployments -n runners

# Verify traffic routing
dig runners.company.com
curl -I https://runners.company.com/health
```

**Prepare Standby Cluster**:
```bash
# Switch to standby (Green)
kubectl config use-context green-cluster

# Ensure cluster healthy
kubectl get nodes
kubectl get pods --all-namespaces
```

### 2. Deployment Phase

**Deploy to Standby (Green) Cluster**:

```bash
# Update infrastructure (Terraform)
cd terraform/green-cluster
terraform plan
terraform apply

# Deploy application updates (Helm)
helm upgrade runner-controller \
  ./charts/runner-controller \
  --namespace runners \
  --values green-values.yaml

# Deploy Karpenter updates
kubectl apply -f karpenter-provisioners/
```

**What Gets Deployed**:
- Kubernetes version updates
- Runner controller updates
- Karpenter configuration changes
- Network policy updates
- New runner images

### 3. Validation Phase

**Smoke Tests**:
```bash
# 1. Cluster health
kubectl get nodes
kubectl get pods -A | grep -v Running

# 2. Runner controller health
kubectl logs -n runners deployment/runner-controller

# 3. Karpenter health
kubectl logs -n karpenter deployment/karpenter

# 4. Network connectivity
kubectl run test-pod --image=busybox --rm -it -- wget -O- https://api.github.com
```

**Integration Tests**:
```bash
# Trigger test workflow
gh workflow run test-runner.yml \
  --repo company/test-repo \
  --ref main

# Monitor job assignment
kubectl logs -f -n runners deployment/runner-controller

# Verify pod creation
kubectl get pods -n runners -l app=github-runner

# Check job completion
gh run list --repo company/test-repo --limit 1
```

**Canary Testing** (Optional):
```bash
# Route 10% traffic to Green cluster
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXX \
  --change-batch file://canary-10pct.json

# Monitor for 15 minutes
watch -n 10 'kubectl top nodes'

# Check error rates
curl https://metrics.company.com/api/errors?cluster=green
```

### 4. Cutover Phase

**Pre-Cutover Checklist**:
- [ ] All validation tests passed
- [ ] Canary testing successful (if used)
- [ ] Rollback plan documented
- [ ] Team available for monitoring
- [ ] Stakeholders notified

**Traffic Switchover**:

```bash
# Lower DNS TTL for faster propagation
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXX \
  --change-batch file://lower-ttl.json

# Wait for TTL to propagate (5 minutes)
sleep 300

# Switch traffic to Green cluster
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXX \
  --change-batch file://switch-to-green.json

# Monitor traffic shift
watch -n 5 'dig +short runners.company.com'
```

**Monitoring During Cutover**:
```bash
# Watch pod creation rate
watch -n 2 'kubectl get pods -n runners | grep -c Running'

# Monitor error logs
kubectl logs -f -n runners deployment/runner-controller | grep ERROR

# Check GitHub job queue
gh api /repos/company/repo/actions/runs \
  --jq '.workflow_runs[] | select(.status=="queued") | .id'

# CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EKS \
  --metric-name node_cpu_utilization \
  --dimensions Name=ClusterName,Value=green-cluster
```

### 5. Validation Post-Cutover

**Verify Green Cluster Active**:
```bash
# Check traffic distribution
dig runners.company.com  # Should show Green LB

# Verify jobs running on Green
kubectl get pods -n runners -o wide
kubectl logs -n runners <runner-pod>

# Check job success rate
gh run list --repo company/repo --limit 10 --json conclusion
```

**Monitor for Issues**:
- Error rates (should be < 1%)
- Job queue depth (should be < 10)
- Pod start times (should be < 10s)
- Node provisioning (should be < 60s)

### 6. Cleanup Phase

**Update Blue Cluster**:
```bash
# Blue is now standby
# Option 1: Keep at full scale for fast rollback
# Option 2: Scale down to save costs

# Scale down worker nodes (if option 2)
kubectl scale deployment -n runners runner-controller --replicas=1
```

**Restore DNS TTL**:
```bash
# Increase TTL back to normal
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXX \
  --change-batch file://restore-ttl.json
```

**Documentation**:
- Update runbooks with new active cluster
- Record deployment in change log
- Document any issues encountered

---

## Rollback Procedure

### When to Rollback

Rollback if:
- Error rate > 5% for 5+ minutes
- Critical functionality broken
- Job queue growing uncontrollably (> 100 jobs)
- Node provisioning failures (> 50%)

### Rollback Steps

**Immediate Rollback** (< 5 minutes):

```bash
# 1. Switch traffic back to Blue
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXX \
  --change-batch file://switch-to-blue.json

# 2. Verify traffic restored
dig +short runners.company.com

# 3. Verify jobs running on Blue
kubectl config use-context blue-cluster
kubectl get pods -n runners

# 4. Monitor error rates
# Should return to normal within 2 minutes
```

**Post-Rollback**:
```bash
# 1. Investigate Green cluster issues
kubectl config use-context green-cluster
kubectl logs -n runners deployment/runner-controller --tail=1000

# 2. Fix issues on Green (now standby)
# Don't rush - take time to diagnose properly

# 3. Re-test deployment process
# Validate fixes before next cutover attempt

# 4. Document root cause and fixes
```

---

## Disaster Recovery

### Failure Scenarios

#### Scenario 1: Active Cluster Total Failure

**Cause**: AWS region outage, cluster corruption, network partition

**Recovery**:
```bash
# 1. Immediate failover to standby
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXX \
  --change-batch file://emergency-failover.json

# 2. Scale up standby cluster (if scaled down)
kubectl config use-context green-cluster
kubectl scale deployment -n runners runner-controller --replicas=10

# 3. Verify traffic routing
dig +short runners.company.com

# 4. Monitor job execution
kubectl get pods -n runners -w
```

**RTO (Recovery Time Objective)**: 5 minutes  
**RPO (Recovery Point Objective)**: 0 (no data loss)

#### Scenario 2: Karpenter Failure

**Cause**: Karpenter controller crash, AWS API throttling

**Recovery**:
```bash
# Active cluster
kubectl rollout restart -n karpenter deployment/karpenter

# If restart fails, failover to standby
# Standby Karpenter will handle scaling
```

#### Scenario 3: Control Plane Failure

**Cause**: Kubernetes API server unavailable

**Recovery**:
```bash
# Control plane failure = cluster unavailable
# Immediate failover to standby cluster
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXX \
  --change-batch file://emergency-failover.json

# AWS will restore control plane (typically < 15 min)
# Standby handles traffic during restoration
```

---

## Operational Procedures

### Weekly Deployment Cadence

**Schedule**: Every Tuesday, 10am PT (low-traffic window)

**Rotation**:
- Week 1: Deploy to Green, cutover to Green (Blue → Green)
- Week 2: Deploy to Blue, cutover to Blue (Green → Blue)
- Repeat

**Benefits of Regular Rotation**:
- Both clusters stay up-to-date
- Regular validation of failover process
- Team stays practiced on procedures

### Emergency Updates

**Criteria for Emergency Deployment**:
- Critical security vulnerability (CVE with active exploits)
- Production incident requiring immediate fix
- Data loss prevention

**Expedited Process**:
1. Deploy to standby cluster (skip canary)
2. Minimum smoke tests only
3. Immediate cutover
4. Enhanced monitoring

**Approval Required**:
- Engineering Director (or higher)
- On-call engineer must be available
- Incident channel created

### Maintenance Windows

**Scheduled Maintenance**: First Saturday of month, 2am-6am PT

**Activities**:
- Kubernetes version upgrades
- Major infrastructure changes
- EKS control plane updates

**Process**:
1. Deploy to standby cluster during maintenance window
2. Extensive testing (no time pressure)
3. Cutover on Tuesday following maintenance
4. Both clusters run identical versions

---

## Cost Optimization

### Standby Cluster Scaling

**Options**:

**Option 1: Full Scale Standby** (Recommended)
- Keep standby at same scale as active
- **Cost**: 2x infrastructure cost
- **Benefit**: Instant failover (< 1 minute)

**Option 2: Reduced Scale Standby**
- Run standby at 25% capacity
- Scale up during cutover (5-10 minutes)
- **Cost**: 1.25x infrastructure cost
- **Benefit**: Cost savings with acceptable RTO

**Option 3: Minimal Standby**
- Single node per pool on standby
- Full scale-up during emergency (10-15 minutes)
- **Cost**: 1.1x infrastructure cost
- **Benefit**: Maximum cost savings
- **Risk**: Slower failover

**Current Configuration**: Option 2 (reduced scale standby)

### Spot Instance Usage

Both clusters use **75% Spot instances** where possible:
- Spot for worker nodes (runner pods tolerate interruptions)
- On-Demand for control plane and critical infrastructure

**Spot Interruption Handling**:
- Karpenter automatically provisions replacement nodes
- Runner pods terminated gracefully (job re-queued)
- Blue/Green provides backup if widespread spot interruptions

---

## Monitoring and Observability

### Key Metrics

**Cluster Health**:
- Node count (active vs standby)
- Pod count per cluster
- Control plane health

**Traffic Distribution**:
- Requests per cluster (CloudWatch)
- Active connections (LB metrics)
- DNS query responses (Route 53)

**Job Execution**:
- Job queue depth per cluster
- Job success/failure rate
- Pod start time (P95, P99)

**Cost**:
- Hourly cost per cluster (AWS Cost Explorer)
- Spot vs On-Demand ratio
- Idle resource percentage

### Alerts

**Critical Alerts** (Page on-call):
- Active cluster control plane unhealthy
- Error rate > 10% for 5 minutes
- Both clusters unhealthy (disaster scenario)

**Warning Alerts** (Slack notification):
- Standby cluster unhealthy
- Deployment validation failed
- Cost anomaly detected (> 20% increase)

---

## Comparison with Other Strategies

| Strategy | Downtime | Rollback | Cost | Complexity |
|----------|----------|----------|------|------------|
| **Blue/Green** | Zero | Instant | 2x | Medium |
| Rolling Update | Minimal | Slow | 1x | Low |
| Canary | Minimal | Fast | 1.1x | High |
| Recreate | Minutes | Manual | 1x | Low |

**Why Blue/Green for This Platform**:
- Zero downtime critical (CI/CD must be always-on)
- Instant rollback needed (developers can't wait)
- Cost acceptable (infrastructure <10% of total DevEx budget)
- Team size supports operational complexity

---

## Best Practices

### Do These Things

✅ **Automate Everything**
- Terraform for infrastructure
- Helm for application deployment
- Scripts for cutover process

✅ **Test Regularly**
- Practice failover monthly
- Run disaster recovery drills quarterly
- Validate standby cluster weekly

✅ **Monitor Closely**
- Dashboards for both clusters
- Alerts for health degradation
- Real-time traffic monitoring during cutover

✅ **Document Thoroughly**
- Runbooks for all procedures
- Post-deployment reports
- Incident retrospectives

✅ **Communicate Proactively**
- Notify teams of maintenance windows
- Post deployment status in Slack
- Share metrics on platform health

### Avoid These Mistakes

❌ **Don't Neglect Standby Cluster**
- Standby must be tested regularly
- Untested standby = not a backup

❌ **Don't Skip Validation**
- Always run smoke tests before cutover
- Integration tests are not optional

❌ **Don't Rush Rollbacks**
- Better to rollback quickly than fix forward slowly
- Rollback threshold should be low

❌ **Don't Ignore Costs**
- Monitor both clusters' costs
- Optimize standby scaling strategy

❌ **Don't Forget DNS TTL**
- Lower TTL before deployment
- Restore TTL after deployment

---

## Conclusion

Blue/Green deployment enables the Ephemeral GitHub Actions Runner Platform to achieve:

✅ **Zero Downtime**: Seamless deployments without user impact  
✅ **Instant Failover**: < 1 minute RTO for disaster recovery  
✅ **Safe Testing**: Validate changes on standby before production  
✅ **Quick Rollback**: Revert to previous environment in seconds  
✅ **High Confidence**: Team practiced on deployment procedures  

While Blue/Green is more expensive and complex than alternatives, it delivers the reliability and availability required for mission-critical CI/CD infrastructure.

---

## Appendices

### Appendix A: Terraform Snippets

```hcl
# Blue cluster
module "blue_cluster" {
  source = "./modules/eks-cluster"
  
  cluster_name = "runners-blue"
  environment  = "production"
  is_active    = true
  
  node_groups = {
    general = { min_size = 10, max_size = 100 }
    gpu     = { min_size = 2,  max_size = 20 }
  }
}

# Green cluster
module "green_cluster" {
  source = "./modules/eks-cluster"
  
  cluster_name = "runners-green"
  environment  = "production"
  is_active    = false  # Standby
  
  node_groups = {
    general = { min_size = 3,  max_size = 100 }  # Reduced scale
    gpu     = { min_size = 1,  max_size = 20 }
  }
}
```

### Appendix B: Route 53 Change Batch

```json
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "runners.company.com",
        "Type": "A",
        "SetIdentifier": "Blue",
        "Weight": 0,
        "AliasTarget": {
          "HostedZoneId": "Z1234",
          "DNSName": "blue-lb.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "runners.company.com",
        "Type": "A",
        "SetIdentifier": "Green",
        "Weight": 100,
        "AliasTarget": {
          "HostedZoneId": "Z5678",
          "DNSName": "green-lb.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }
  ]
}
```

### Appendix C: Deployment Checklist

```markdown
## Pre-Deployment
- [ ] Identify active and standby clusters
- [ ] Verify standby cluster health
- [ ] Lower DNS TTL to 60 seconds
- [ ] Notify #engineering of deployment
- [ ] Ensure on-call engineer available

## Deployment
- [ ] Deploy infrastructure (Terraform)
- [ ] Deploy application (Helm)
- [ ] Run smoke tests
- [ ] Run integration tests
- [ ] Optional: Canary 10% traffic
- [ ] Monitor for 15 minutes

## Cutover
- [ ] Pre-cutover checklist complete
- [ ] Switch DNS routing to standby
- [ ] Monitor traffic shift
- [ ] Verify jobs running on new cluster
- [ ] Check error rates < 1%

## Post-Deployment
- [ ] Restore DNS TTL to 300 seconds
- [ ] Update documentation (active cluster)
- [ ] Post deployment status to Slack
- [ ] Record deployment in change log
```